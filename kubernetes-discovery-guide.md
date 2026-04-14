# Kubernetes 서비스 디스커버리 가이드

Alloy는 Kubernetes API를 통해 스크랩 대상을 자동으로 발견합니다.
`discovery.kubernetes` 컴포넌트가 핵심입니다.

---

## discovery.kubernetes 기본

```alloy
discovery.kubernetes "pods" {
  role = "pod"   // pod, node, service, endpoint, endpointslice, ingress
}
```

### role 종류

| role | 설명 | 주요 메타라벨 |
|---|---|---|
| `pod` | 클러스터의 모든 Pod | `__meta_kubernetes_pod_*` |
| `node` | 클러스터의 모든 Node | `__meta_kubernetes_node_*` |
| `service` | 클러스터의 모든 Service | `__meta_kubernetes_service_*` |
| `endpoint` | Service의 Endpoints | `__meta_kubernetes_endpoint_*` |
| `endpointslice` | EndpointSlice 리소스 | `__meta_kubernetes_endpointslice_*` |
| `ingress` | 클러스터의 모든 Ingress | `__meta_kubernetes_ingress_*` |

---

## 네임스페이스 필터링

```alloy
// 특정 네임스페이스만 대상
discovery.kubernetes "app_pods" {
  role = "pod"

  namespaces {
    names = ["production", "staging"]
  }
}

// 모든 네임스페이스 (기본값)
discovery.kubernetes "all_pods" {
  role = "pod"
  // namespaces 블록 생략 시 전체 탐색
}

// Alloy 자신의 네임스페이스만
discovery.kubernetes "same_ns" {
  role = "pod"
  namespaces {
    own_namespace = true
  }
}
```

---

## 레이블 셀렉터 필터링

```alloy
// app=myapp 레이블이 있는 Pod만
discovery.kubernetes "myapp_pods" {
  role = "pod"

  selectors {
    role  = "pod"
    label = "app=myapp"
  }
}

// 여러 셀렉터 조합
discovery.kubernetes "monitored_pods" {
  role = "pod"

  selectors {
    role  = "pod"
    label = "monitoring=true,environment=production"
  }
}
```

---

## discovery.relabel로 메타라벨 처리

`discovery.kubernetes`가 반환하는 타겟에는 `__meta_kubernetes_*` 형태의 메타라벨이 포함됩니다.
`discovery.relabel`로 이를 정리합니다.

### 주요 메타라벨 (Pod)

```
__meta_kubernetes_namespace                    → 네임스페이스
__meta_kubernetes_pod_name                     → Pod 이름
__meta_kubernetes_pod_label_<KEY>              → Pod 레이블
__meta_kubernetes_pod_annotation_<KEY>         → Pod 어노테이션
__meta_kubernetes_pod_ip                       → Pod IP
__meta_kubernetes_pod_container_name           → 컨테이너 이름
__meta_kubernetes_pod_container_port_number    → 컨테이너 포트 번호
__meta_kubernetes_pod_node_name                → Pod가 실행 중인 노드
__meta_kubernetes_pod_ready                    → Pod Ready 상태
__meta_kubernetes_pod_phase                    → Pod Phase (Running/Pending/...)
```

### 릴레이블링 예시

```alloy
discovery.relabel "pods" {
  targets = discovery.kubernetes.pods.targets

  // 네임스페이스를 레이블로 추가
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  // Pod 이름을 레이블로 추가
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  // 컨테이너 이름을 레이블로 추가
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }

  // 노드 이름을 레이블로 추가
  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }

  // 어노테이션으로 스크랩 포트 동적 지정
  // prometheus.io/port: "8080" 어노테이션이 있으면 해당 포트 사용
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_port"]
    regex         = "(.+)"
    target_label  = "__address__"
    replacement   = "${1}"
    action        = "replace"
  }
}
```

---

## 어노테이션 기반 스크랩 제어

Pod에 어노테이션을 달아 스크랩 여부와 설정을 동적으로 제어합니다.

### Pod 어노테이션 예시

```yaml
# Pod spec
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
    prometheus.io/scheme: "http"
```

### Alloy에서 어노테이션 기반 필터링

```alloy
discovery.kubernetes "pods" {
  role = "pod"
}

discovery.relabel "annotated_pods" {
  targets = discovery.kubernetes.pods.targets

  // prometheus.io/scrape=true 어노테이션 있는 Pod만 통과
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_scrape"]
    regex         = "true"
    action        = "keep"
  }

  // prometheus.io/path 어노테이션으로 metrics 경로 설정
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_path"]
    regex         = "(.+)"
    target_label  = "__metrics_path__"
  }

  // prometheus.io/port로 주소 재설정
  rule {
    source_labels = ["__address__", "__meta_kubernetes_pod_annotation_prometheus_io_port"]
    regex         = "([^:]+)(?::\\d+)?;(\\d+)"
    replacement   = "$1:$2"
    target_label  = "__address__"
  }

  // 레이블 정리
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
}
```

---

## PodMonitor & ServiceMonitor 지원

Prometheus Operator의 CRD를 그대로 활용할 수 있습니다.

```alloy
// PodMonitor 자동 인식
prometheus.operator.podmonitors "default" {
  forward_to = [prometheus.remote_write.mimir.receiver]

  // 특정 네임스페이스만
  namespaces = ["production", "monitoring"]
}

// ServiceMonitor 자동 인식
prometheus.operator.servicemonitors "default" {
  forward_to = [prometheus.remote_write.mimir.receiver]
}
```

이 방식을 사용하면 개별 어노테이션 없이 PodMonitor/ServiceMonitor 리소스로 스크랩을 선언적으로 관리할 수 있습니다.

---

## Node 메트릭 수집

```alloy
// 노드 kubelet 메트릭 수집
discovery.kubernetes "nodes" {
  role = "node"
}

discovery.relabel "nodes" {
  targets = discovery.kubernetes.nodes.targets

  // HTTPS 스킴으로 변경 (kubelet은 HTTPS)
  rule {
    target_label = "__scheme__"
    replacement  = "https"
  }

  // kubelet 메트릭 경로
  rule {
    target_label  = "__metrics_path__"
    replacement   = "/metrics"
  }

  // 노드 이름 레이블
  rule {
    source_labels = ["__meta_kubernetes_node_name"]
    target_label  = "node"
  }
}

prometheus.scrape "kubelet" {
  targets    = discovery.relabel.nodes.output
  forward_to = [prometheus.remote_write.mimir.receiver]

  scheme = "https"

  tls_config {
    ca_file              = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    insecure_skip_verify = false
  }

  bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
}
```

---

## 클러스터 수준 리소스 발견

```alloy
// 서비스 디스커버리 (Service 기반)
discovery.kubernetes "services" {
  role = "service"

  selectors {
    role  = "service"
    label = "app.kubernetes.io/managed-by=Helm"
  }
}

// Endpoint 기반 (Service의 실제 Pod 엔드포인트)
discovery.kubernetes "endpoints" {
  role = "endpoint"
}

discovery.relabel "endpoints" {
  targets = discovery.kubernetes.endpoints.targets

  // Running Pod의 엔드포인트만 유지
  rule {
    source_labels = ["__meta_kubernetes_pod_phase"]
    regex         = "Running"
    action        = "keep"
  }

  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  rule {
    source_labels = ["__meta_kubernetes_service_name"]
    target_label  = "service"
  }
}
```

---

## 다음 단계

- [메트릭 수집 파이프라인](./metrics-guide.md)
- [로그 수집 파이프라인](./logs-guide.md)
