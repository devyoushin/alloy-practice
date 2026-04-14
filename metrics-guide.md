# 메트릭 수집 파이프라인 가이드

Alloy로 Kubernetes 클러스터의 메트릭을 수집하여 Mimir(또는 다른 Prometheus 호환 백엔드)로 전송합니다.

---

## 전체 파이프라인 구조

```
discovery.kubernetes ──▶ discovery.relabel ──▶ prometheus.scrape ──▶ prometheus.remote_write
       (대상 발견)            (필터/변환)             (스크랩)               (Mimir 전송)
```

---

## prometheus.remote_write 설정

```alloy
prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir-nginx.monitoring.svc.cluster.local:8080/api/v1/push"

    // 멀티 테넌트 사용 시 (Mimir X-Scope-OrgID 헤더)
    headers = {
      "X-Scope-OrgID" = "myteam",
    }

    // 재시도 및 큐 설정
    queue_config {
      capacity             = 10000
      max_shards           = 50
      min_shards           = 1
      max_samples_per_send = 2000
      batch_send_deadline  = "5s"
      min_backoff          = "30ms"
      max_backoff          = "5s"
    }
  }

  // WAL 설정 (재시작 후 데이터 복구)
  wal {
    truncate_frequency = "2h"
    min_keepalive_time = "5m"
    max_keepalive_time = "8h"
  }

  // 외부 레이블 추가 (모든 메트릭에 공통 레이블)
  external_labels = {
    cluster = "production-eks",
    region  = "ap-northeast-2",
  }
}
```

---

## prometheus.scrape 설정

```alloy
prometheus.scrape "pods" {
  targets         = discovery.relabel.pods.output
  forward_to      = [prometheus.remote_write.mimir.receiver]

  scrape_interval = "30s"
  scrape_timeout  = "10s"

  // 메트릭 경로 (기본값: /metrics)
  metrics_path = "/metrics"

  // Job 레이블
  job_name = "kubernetes-pods"

  // 네이티브 히스토그램 지원 (Prometheus 2.40+)
  scrape_protocols              = ["OpenMetricsText1.0.0", "OpenMetricsText0.0.1", "PrometheusText0.0.4"]
  enable_protobuf_negotiation   = true
}
```

---

## 전체 메트릭 수집 설정 예시

### Pod 메트릭 수집

```alloy
// 1. 모든 Pod 발견
discovery.kubernetes "pods" {
  role = "pod"
}

// 2. 어노테이션 기반 필터링 및 레이블 정리
discovery.relabel "pods" {
  targets = discovery.kubernetes.pods.targets

  // prometheus.io/scrape=true인 Pod만 수집
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_scrape"]
    regex         = "true"
    action        = "keep"
  }

  // Running 상태의 Pod만 수집
  rule {
    source_labels = ["__meta_kubernetes_pod_phase"]
    regex         = "Running"
    action        = "keep"
  }

  // 커스텀 포트 어노테이션 적용
  rule {
    source_labels = ["__address__", "__meta_kubernetes_pod_annotation_prometheus_io_port"]
    regex         = "([^:]+)(?::\\d+)?;(\\d+)"
    replacement   = "$1:$2"
    target_label  = "__address__"
  }

  // 커스텀 메트릭 경로 어노테이션 적용
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_path"]
    regex         = "(.+)"
    target_label  = "__metrics_path__"
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

  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }
}

// 3. 스크랩
prometheus.scrape "pods" {
  targets         = discovery.relabel.pods.output
  forward_to      = [prometheus.remote_write.mimir.receiver]
  scrape_interval = "30s"
  job_name        = "kubernetes-pods"
}
```

### Node (kubelet) 메트릭 수집

```alloy
discovery.kubernetes "nodes" {
  role = "node"
}

discovery.relabel "nodes" {
  targets = discovery.kubernetes.nodes.targets

  rule {
    target_label = "__scheme__"
    replacement  = "https"
  }

  rule {
    target_label = "__metrics_path__"
    replacement  = "/metrics"
  }

  rule {
    source_labels = ["__meta_kubernetes_node_name"]
    target_label  = "node"
  }
}

prometheus.scrape "kubelet" {
  targets    = discovery.relabel.nodes.output
  forward_to = [prometheus.remote_write.mimir.receiver]

  scheme   = "https"
  job_name = "kubelet"

  tls_config {
    ca_file              = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    insecure_skip_verify = false
  }

  bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
}
```

### cAdvisor 메트릭 수집 (컨테이너 리소스)

```alloy
discovery.relabel "cadvisor" {
  targets = discovery.kubernetes.nodes.targets

  rule {
    target_label = "__scheme__"
    replacement  = "https"
  }

  rule {
    target_label = "__metrics_path__"
    replacement  = "/metrics/cadvisor"
  }

  rule {
    source_labels = ["__meta_kubernetes_node_name"]
    target_label  = "node"
  }
}

prometheus.scrape "cadvisor" {
  targets    = discovery.relabel.cadvisor.output
  forward_to = [prometheus.remote_write.mimir.receiver]

  scheme   = "https"
  job_name = "cadvisor"

  tls_config {
    ca_file              = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    insecure_skip_verify = false
  }

  bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
}
```

---

## 메트릭 필터링 (prometheus.relabel)

수집 후 불필요한 메트릭을 제거합니다.

```alloy
prometheus.relabel "filter" {
  forward_to = [prometheus.remote_write.mimir.receiver]

  // 특정 메트릭만 유지 (allowlist)
  rule {
    source_labels = ["__name__"]
    regex         = "container_(cpu|memory|network).*"
    action        = "keep"
  }

  // 특정 메트릭 제외 (denylist)
  rule {
    source_labels = ["__name__"]
    regex         = "go_gc_.*"
    action        = "drop"
  }

  // 특정 레이블 삭제
  rule {
    regex  = "tmp_.*"
    action = "labeldrop"
  }
}

prometheus.scrape "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [prometheus.relabel.filter.receiver]  // relabel 거친 후 전송
}
```

---

## PodMonitor / ServiceMonitor 활용

기존 Prometheus Operator CRD를 그대로 사용합니다.

```alloy
// PodMonitor 자동 인식
prometheus.operator.podmonitors "all" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}

// ServiceMonitor 자동 인식
prometheus.operator.servicemonitors "all" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}
```

```yaml
# 기존 PodMonitor 리소스 (변경 없이 재사용 가능)
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app
  namespace: production
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## 다음 단계

- [로그 수집 파이프라인](./logs-guide.md)
- [트레이스 수집 파이프라인](./traces-guide.md)
