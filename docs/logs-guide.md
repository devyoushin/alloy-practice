# 로그 수집 파이프라인 가이드

Alloy로 Kubernetes Pod 로그를 수집하여 Loki로 전송합니다.
DaemonSet으로 배포되어 각 노드의 컨테이너 로그에 직접 접근합니다.

---

## 전체 파이프라인 구조

```
loki.source.kubernetes ──▶ loki.relabel ──▶ loki.write
    (Pod 로그 수집)          (레이블 정리)      (Loki 전송)
```

또는 파일 기반:

```
discovery.kubernetes ──▶ loki.source.kubernetes_logs ──▶ loki.write
```

---

## loki.write 설정

```alloy
loki.write "loki" {
  endpoint {
    url = "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push"

    // 멀티 테넌트 (Loki X-Scope-OrgID 헤더)
    headers = {
      "X-Scope-OrgID" = "myteam",
    }

    // 재시도 설정
    backoff_config {
      min_period  = "500ms"
      max_period  = "5m"
      max_retries = 10
    }
  }

  // 외부 레이블 (모든 로그 스트림에 공통 레이블)
  external_labels = {
    cluster = "production-eks",
    region  = "ap-northeast-2",
  }
}
```

---

## loki.source.kubernetes 설정

Kubernetes API를 통해 Pod 로그를 직접 스트리밍합니다.

```alloy
// 1. Pod 발견
discovery.kubernetes "pods" {
  role = "pod"
}

// 2. 릴레이블링
discovery.relabel "pods" {
  targets = discovery.kubernetes.pods.targets

  // Running 상태의 Pod만
  rule {
    source_labels = ["__meta_kubernetes_pod_phase"]
    regex         = "Running"
    action        = "keep"
  }

  // 네임스페이스 레이블
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  // Pod 이름 레이블
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  // 컨테이너 이름 레이블
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }

  // app 레이블
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "app"
  }

  // 노드 이름 레이블
  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }
}

// 3. 로그 수집
loki.source.kubernetes "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [loki.write.loki.receiver]
}
```

---

## 파일 기반 로그 수집 (DaemonSet)

DaemonSet 배포 시 노드의 `/var/log/pods` 디렉토리를 직접 읽습니다.

```alloy
// 1. Pod 정보 발견 (파일 경로 매핑용)
discovery.kubernetes "pods" {
  role = "pod"
}

// 2. 파일 경로 매핑
discovery.relabel "pod_logs" {
  targets = discovery.kubernetes.pods.targets

  // 로그 파일 경로 설정
  // /var/log/pods/<namespace>_<pod_name>_<uid>/<container_name>/*.log
  rule {
    source_labels = [
      "__meta_kubernetes_pod_uid",
      "__meta_kubernetes_pod_container_name",
    ]
    separator    = "/"
    target_label = "__path__"
    replacement  = "/var/log/pods/*$1/*.log"
  }

  // 레이블 추가
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "app"
  }
}

// 3. 파일 로그 수집
loki.source.file "pod_logs" {
  targets    = discovery.relabel.pod_logs.output
  forward_to = [loki.write.loki.receiver]

  // 위치 파일 (재시작 후 이어서 읽기)
  legacy_positions_file = "/var/lib/alloy/positions.yaml"
}
```

---

## 로그 파싱 (loki.process)

```alloy
loki.process "parse_json" {
  forward_to = [loki.write.loki.receiver]

  // JSON 파싱
  stage.json {
    expressions = {
      level     = "level",
      timestamp = "time",
      message   = "msg",
      trace_id  = "trace_id",
    }
  }

  // 파싱된 필드를 레이블로 추가
  stage.labels {
    values = {
      level = "",
    }
  }

  // 타임스탬프 재설정
  stage.timestamp {
    source = "timestamp"
    format = "RFC3339"
  }
}

loki.source.kubernetes "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [loki.process.parse_json.receiver]  // 파싱 후 전송
}
```

---

## 로그 필터링

```alloy
loki.process "filter" {
  forward_to = [loki.write.loki.receiver]

  // 특정 패턴 제외 (불필요한 헬스체크 로그 등)
  stage.drop {
    expression       = ".*health.*check.*"
    drop_counter_reason = "health_check_log"
  }

  // 특정 네임스페이스 로그만 유지
  stage.match {
    selector = "{namespace=\"production\"}"
    // 매칭 시 이후 stage 실행
    stage.json {
      expressions = {
        level = "level",
      }
    }
  }

  // 로그 레벨 필터링 (ERROR 이상만)
  stage.drop {
    source     = "level"
    expression = "debug|info|warn"
  }
}
```

---

## 멀티라인 로그 처리

Java 스택 트레이스, Python traceback 등 여러 줄로 이루어진 로그 처리:

```alloy
loki.process "multiline" {
  forward_to = [loki.write.loki.receiver]

  // Java 스택 트레이스 병합
  stage.multiline {
    firstline     = "^\\d{4}-\\d{2}-\\d{2}"  // 날짜로 시작하는 줄이 새 로그
    max_wait_time = "3s"
    max_lines     = 128
  }
}
```

---

## 시스템 로그 수집

```alloy
// /var/log/syslog 등 노드 시스템 로그
local.file_match "syslog" {
  path_targets = [
    { __path__ = "/var/log/syslog", job = "syslog", node = env("NODE_NAME") },
    { __path__ = "/var/log/kern.log", job = "kernel", node = env("NODE_NAME") },
  ]
}

loki.source.file "syslog" {
  targets    = local.file_match.syslog.targets
  forward_to = [loki.write.loki.receiver]
}
```

---

## Loki Push API 직접 수신

앱에서 직접 Alloy로 로그를 Push합니다.

```alloy
// HTTP 수신 엔드포인트 (Loki Push API)
loki.source.api "push" {
  http {
    listen_address = "0.0.0.0"
    listen_port    = 3100
  }

  forward_to = [loki.write.loki.receiver]
}
```

---

## 다음 단계

- [트레이스 수집 파이프라인](./traces-guide.md)
- [OpenTelemetry 통합](./otel-guide.md)
