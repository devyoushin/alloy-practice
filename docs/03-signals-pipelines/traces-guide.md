# 트레이스 수집 파이프라인 가이드

Alloy로 분산 트레이싱 데이터를 수집하여 Tempo로 전송합니다.
OpenTelemetry Protocol(OTLP)이 기본 통신 방식입니다.

---

## 전체 파이프라인 구조

```
앱 (OTLP Push)
      │
      ▼
otelcol.receiver.otlp
      │
      ▼
otelcol.processor.batch      ─▶ (선택) otelcol.processor.memory_limiter
      │
      ▼
otelcol.exporter.otlp ──▶ Tempo
```

---

## otelcol.receiver.otlp 설정

앱에서 OTLP 형식으로 트레이스를 Push할 엔드포인트를 엽니다.

```alloy
otelcol.receiver.otlp "default" {
  // gRPC 수신 (기본 포트: 4317)
  grpc {
    endpoint = "0.0.0.0:4317"
  }

  // HTTP 수신 (기본 포트: 4318)
  http {
    endpoint = "0.0.0.0:4318"
  }

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}
```

---

## otelcol.processor.batch 설정

효율적인 전송을 위해 데이터를 배치로 묶습니다.

```alloy
otelcol.processor.batch "default" {
  // 최대 대기 시간
  timeout = "5s"

  // 배치당 최대 span 수
  send_batch_size = 1000

  // 배치당 최대 크기 (bytes)
  send_batch_max_size = 10000

  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}
```

---

## otelcol.processor.memory_limiter (선택)

메모리 과다 사용 방지:

```alloy
otelcol.processor.memory_limiter "default" {
  // 메모리 사용량이 이 값을 넘으면 데이터 drop
  limit_mib = 512

  // 스파이크 허용 추가 메모리
  spike_limit_mib = 128

  // 체크 주기
  check_interval = "1s"

  output {
    traces = [otelcol.processor.batch.default.input]
  }
}
```

---

## otelcol.exporter.otlp 설정

Tempo로 트레이스를 전송합니다.

```alloy
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo.monitoring.svc.cluster.local:4317"

    // TLS 비활성화 (클러스터 내부 통신)
    tls {
      insecure = true
    }

    // 멀티 테넌트
    headers = {
      "X-Scope-OrgID" = "myteam",
    }

    // 재시도 설정
    retry_on_failure {
      enabled          = true
      initial_interval = "5s"
      max_interval     = "30s"
      max_elapsed_time = "5m"
    }

    // 큐 설정
    sending_queue {
      enabled    = true
      num_consumers = 10
      queue_size = 1000
    }
  }
}
```

---

## HTTP로 전송하는 경우

```alloy
otelcol.exporter.otlphttp "tempo" {
  client {
    endpoint = "http://tempo.monitoring.svc.cluster.local:4318"

    tls {
      insecure = true
    }
  }
}
```

---

## 전체 트레이스 파이프라인 예시

```alloy
// 1. OTLP 수신
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }
  output {
    traces = [otelcol.processor.memory_limiter.default.input]
  }
}

// 2. 메모리 제한
otelcol.processor.memory_limiter "default" {
  limit_mib       = 512
  spike_limit_mib = 128
  check_interval  = "1s"
  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

// 3. 배치 처리
otelcol.processor.batch "default" {
  timeout         = "5s"
  send_batch_size = 1000
  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}

// 4. Tempo 전송
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo.monitoring.svc.cluster.local:4317"
    tls {
      insecure = true
    }
  }
}
```

---

## 트레이스에서 메트릭 생성 (Span Metrics)

트레이스 데이터로부터 RED 메트릭(Rate, Error, Duration)을 자동 생성합니다.

```alloy
otelcol.connector.spanmetrics "default" {
  histogram {
    explicit {
      buckets = ["100ms", "250ms", "500ms", "1s", "2s", "5s"]
    }
  }

  dimensions = [
    { name = "http.method" },
    { name = "http.status_code" },
    { name = "service.name" },
  ]

  output {
    metrics = [otelcol.exporter.prometheus.default.input]
  }
}

otelcol.exporter.prometheus "default" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}

// 파이프라인 연결
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }
  output {
    traces = [
      otelcol.processor.batch.default.input,
      otelcol.connector.spanmetrics.default.input,  // span metrics 생성
    ]
  }
}
```

---

## 샘플링 (Tail-based Sampling)

트레이스 볼륨이 클 경우 중요한 트레이스만 선별합니다.

```alloy
otelcol.processor.tail_sampling "default" {
  // 결정 대기 시간 (모든 span 수집 후 판단)
  decision_wait = "10s"
  num_traces    = 50000

  policy {
    name = "errors-policy"
    type = "status_code"
    status_code {
      status_codes = ["ERROR"]
    }
  }

  policy {
    name = "slow-traces-policy"
    type = "latency"
    latency {
      threshold_ms = 500  // 500ms 이상인 트레이스
    }
  }

  policy {
    name = "sample-10-percent"
    type = "probabilistic"
    probabilistic {
      sampling_percentage = 10
    }
  }

  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}
```

---

## 앱 SDK 설정 (참고)

앱에서 Alloy로 트레이스를 전송하는 SDK 설정:

### Go

```go
exporter, _ := otlptracegrpc.New(context.Background(),
    otlptracegrpc.WithEndpoint("alloy.monitoring.svc.cluster.local:4317"),
    otlptracegrpc.WithInsecure(),
)
```

### Python

```python
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(
    endpoint="alloy.monitoring.svc.cluster.local:4317",
    insecure=True,
)
```

### Java (Spring Boot)

```yaml
# application.yaml
management:
  otlp:
    tracing:
      endpoint: http://alloy.monitoring.svc.cluster.local:4318/v1/traces
```

---

## 다음 단계

- [OpenTelemetry 통합](./otel-guide.md)
- [클러스터링 & 고가용성](../02-architecture-config/clustering-guide.md)
