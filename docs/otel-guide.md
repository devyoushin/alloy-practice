# OpenTelemetry 통합 가이드

Alloy는 OpenTelemetry Collector를 완전히 내장하고 있어 OTel 파이프라인을 네이티브로 지원합니다.
메트릭, 로그, 트레이스를 모두 OTLP로 수신하고, 각 백엔드(Mimir/Loki/Tempo)로 변환하여 전송할 수 있습니다.

---

## OTel vs Prometheus/Loki 컴포넌트 선택 기준

| 상황 | 권장 컴포넌트 |
|---|---|
| Kubernetes Pod 메트릭 수집 (Pull) | `prometheus.scrape` |
| 앱에서 OTLP Push 메트릭 수신 | `otelcol.receiver.otlp` → `otelcol.exporter.prometheus` |
| Kubernetes 로그 수집 | `loki.source.kubernetes` |
| 앱에서 OTLP Push 로그 수신 | `otelcol.receiver.otlp` → `otelcol.exporter.loki` |
| 트레이스 수집 | `otelcol.receiver.otlp` → `otelcol.exporter.otlp` |
| 기존 OTel Collector 설정 마이그레이션 | `otelcol.*` 컴포넌트 |

---

## 통합 OTLP 수신 파이프라인

앱이 메트릭 + 로그 + 트레이스를 한 번에 OTLP로 Push하는 경우:

```alloy
// OTLP 수신 (메트릭 + 로그 + 트레이스 동시 수신)
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    metrics = [otelcol.processor.batch.metrics.input]
    logs    = [otelcol.processor.batch.logs.input]
    traces  = [otelcol.processor.batch.traces.input]
  }
}

// 배치 처리 (메트릭)
otelcol.processor.batch "metrics" {
  timeout = "5s"
  output {
    metrics = [otelcol.exporter.prometheus.default.input]
  }
}

// 배치 처리 (로그)
otelcol.processor.batch "logs" {
  timeout = "5s"
  output {
    logs = [otelcol.exporter.loki.default.input]
  }
}

// 배치 처리 (트레이스)
otelcol.processor.batch "traces" {
  timeout = "5s"
  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}

// 메트릭 → Prometheus remote_write
otelcol.exporter.prometheus "default" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}

// 로그 → Loki
otelcol.exporter.loki "default" {
  forward_to = [loki.write.loki.receiver]
}

// 트레이스 → Tempo
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo.monitoring.svc.cluster.local:4317"
    tls {
      insecure = true
    }
  }
}

// Mimir remote_write 엔드포인트
prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir-nginx.monitoring.svc.cluster.local:8080/api/v1/push"
  }
}

// Loki write 엔드포인트
loki.write "loki" {
  endpoint {
    url = "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push"
  }
}
```

---

## OTel 속성 → Prometheus 레이블 변환

```alloy
otelcol.exporter.prometheus "default" {
  forward_to = [prometheus.remote_write.mimir.receiver]

  // OTel resource attribute를 prometheus 레이블로 변환
  include_scope_info    = false
  include_scope_labels  = false

  // 레이블로 포함할 OTel 속성
  resource_to_telemetry_conversion {
    enabled = true
  }
}
```

OTel 속성 → Prometheus 레이블 자동 변환:
- `service.name` → `service_name`
- `k8s.namespace.name` → `k8s_namespace_name`
- `k8s.pod.name` → `k8s_pod_name`

---

## otelcol.processor.attributes (속성 조작)

```alloy
otelcol.processor.attributes "enrich" {
  // 속성 추가
  action {
    key    = "cluster"
    value  = "production-eks"
    action = "insert"
  }

  // 속성 업데이트
  action {
    key    = "environment"
    value  = "production"
    action = "update"
  }

  // 속성 삭제
  action {
    key    = "http.user_agent"
    action = "delete"
  }

  // 속성 해시 (PII 마스킹)
  action {
    key    = "user.email"
    action = "hash"
  }

  output {
    traces = [otelcol.processor.batch.traces.input]
  }
}
```

---

## otelcol.processor.resource (리소스 속성 조작)

```alloy
otelcol.processor.resource "k8s_metadata" {
  // Kubernetes 메타데이터 추가
  attributes = [
    {
      key    = "k8s.cluster.name"
      value  = "production-eks"
      action = "insert"
    },
  ]

  output {
    traces = [otelcol.processor.batch.traces.input]
  }
}
```

---

## otelcol.processor.transform (데이터 변환)

OTTL(OpenTelemetry Transformation Language) 사용:

```alloy
otelcol.processor.transform "normalize" {
  error_mode = "ignore"

  trace_statements {
    context    = "span"
    statements = [
      // HTTP 상태 코드가 400 이상이면 span 에러 표시
      "set(status.code, STATUS_CODE_ERROR) where attributes[\"http.status_code\"] >= 400",
      // span 이름 정규화
      "replace_pattern(name, \"^GET /api/v[0-9]+/\", \"GET /api/\")",
    ]
  }

  output {
    traces = [otelcol.processor.batch.traces.input]
  }
}
```

---

## Jaeger / Zipkin 수신 (레거시 지원)

기존 Jaeger/Zipkin 기반 앱을 Alloy로 마이그레이션할 때:

```alloy
// Jaeger 수신
otelcol.receiver.jaeger "default" {
  protocols {
    grpc {
      endpoint = "0.0.0.0:14250"
    }
    thrift_http {
      endpoint = "0.0.0.0:14268"
    }
    thrift_compact {
      endpoint = "0.0.0.0:6831"
    }
  }
  output {
    traces = [otelcol.processor.batch.traces.input]
  }
}

// Zipkin 수신
otelcol.receiver.zipkin "default" {
  endpoint = "0.0.0.0:9411"
  output {
    traces = [otelcol.processor.batch.traces.input]
  }
}
```

---

## 기존 OTel Collector config.yaml 마이그레이션

기존 OTel Collector 설정을 Alloy로 변환하는 패턴:

```yaml
# 기존 OTel Collector config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
processors:
  batch:
    timeout: 5s
exporters:
  otlp:
    endpoint: tempo:4317
    tls:
      insecure: true
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

```alloy
// 동등한 Alloy 설정
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

otelcol.processor.batch "default" {
  timeout = "5s"
  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}

otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo:4317"
    tls { insecure = true }
  }
}
```

---

## 다음 단계

- [클러스터링 & 고가용성](./clustering-guide.md)
- [Alloy 자체 모니터링](./monitoring-guide.md)
