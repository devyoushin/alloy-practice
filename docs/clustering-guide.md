# 클러스터링 & 고가용성 가이드

Alloy 클러스터링을 통해 여러 Alloy 인스턴스가 스크랩 타겟을 자동으로 분산 처리합니다.
단일 Alloy의 부하를 줄이고, 장애 시 자동 재분배가 이루어집니다.

---

## 클러스터링이란?

```
클러스터링 없음 (각 인스턴스가 전체 타겟 스크랩):
Pod1: [Alloy A] ─▶ Target 1,2,3,4,5,6
Pod2: [Alloy B] ─▶ Target 1,2,3,4,5,6  (중복 수집!)

클러스터링 활성화 (타겟 자동 분산):
Pod1: [Alloy A] ─▶ Target 1,2,3
Pod2: [Alloy B] ─▶ Target 4,5,6  (각자 담당 타겟만!)
```

Alloy는 **일관된 해싱(Consistent Hashing)** 으로 타겟을 분산합니다.
인스턴스가 추가/제거되면 타겟이 자동으로 재분배됩니다.

---

## alloy.clustering 설정

```alloy
// 클러스터링 활성화
alloy.clustering "default" {
  enabled = true
}

// prometheus.scrape에 클러스터링 적용
prometheus.scrape "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [prometheus.remote_write.mimir.receiver]

  // 클러스터링 활성화된 경우 자동으로 타겟 분산
  clustering {
    enabled = true
  }
}
```

---

## Helm values-ha.yaml 설정

```yaml
# 고가용성 배포 설정
controller:
  type: deployment      # DaemonSet 대신 Deployment 사용
  replicas: 3           # 3개 인스턴스

clustering:
  enabled: true
  name: "alloy-cluster"

service:
  clusterIP: "None"     # Headless Service (Pod 간 직접 통신)

serviceAccount:
  create: true
```

---

## 클러스터 구성 확인

```bash
# Pod 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy

# 클러스터 멤버 확인 (Web UI)
kubectl port-forward svc/alloy 12345:12345 -n monitoring
curl http://localhost:12345/api/v0/component/alloy.clustering.default
```

---

## 클러스터링 + 메트릭 파이프라인 전체 예시

```alloy
// 클러스터링 활성화
alloy.clustering "default" {
  enabled = true
}

// Pod 발견
discovery.kubernetes "pods" {
  role = "pod"
}

// 릴레이블링
discovery.relabel "pods" {
  targets = discovery.kubernetes.pods.targets

  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_scrape"]
    regex         = "true"
    action        = "keep"
  }

  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
}

// 클러스터링 적용된 스크랩
prometheus.scrape "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [prometheus.remote_write.mimir.receiver]

  scrape_interval = "30s"

  clustering {
    enabled = true  // 타겟을 인스턴스 간 자동 분산
  }
}

// Mimir 전송
prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir-nginx.monitoring.svc.cluster.local:8080/api/v1/push"
  }

  external_labels = {
    cluster = "production-eks",
  }
}
```

---

## DaemonSet vs Deployment 선택

| 항목 | DaemonSet | Deployment (클러스터링) |
|---|---|---|
| **배포 방식** | 노드당 1개 | 지정한 replica 수 |
| **용도** | 로그 수집 (노드 파일 접근) | 메트릭 수집 (API 기반) |
| **타겟 분산** | 노드 담당 로그만 | 클러스터링으로 자동 분산 |
| **스케일링** | 노드 수에 종속 | replica 수 독립적 조정 |
| **HA** | 노드 장애 시 해당 노드 로그 유실 | replica 장애 시 자동 재분배 |

**권장 패턴**: 메트릭+트레이스는 Deployment+클러스터링, 로그는 DaemonSet을 별도로 배포

---

## WAL (Write-Ahead Log)

클러스터링과 무관하게 데이터 유실 방지를 위한 WAL 설정:

```alloy
prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir-nginx.monitoring.svc.cluster.local:8080/api/v1/push"

    queue_config {
      capacity             = 10000
      max_shards           = 50
      max_samples_per_send = 2000
    }
  }

  // WAL 설정
  wal {
    // WAL 트런케이션 주기
    truncate_frequency = "2h"

    // 최소 보존 시간 (백엔드 장애 시 재전송용)
    min_keepalive_time = "5m"

    // 최대 보존 시간
    max_keepalive_time = "8h"
  }
}
```

WAL 저장 위치는 Helm values에서 설정:

```yaml
# ../ops/config/helm/values-ha.yaml
alloy:
  extraVolumes:
    - name: wal
      emptyDir: {}
  extraVolumeMounts:
    - name: wal
      mountPath: /var/lib/alloy/wal
  configMap:
    content: |
      prometheus.remote_write "mimir" {
        wal_dir = "/var/lib/alloy/wal"
        ...
      }
```

StatefulSet으로 영구 WAL 보장:

```yaml
controller:
  type: statefulset
  replicas: 3

volumeClaimTemplates:
  - metadata:
      name: wal
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

## 장애 시나리오 대응

### Pod 장애

```
Alloy A (장애)
Alloy B ─▶ A의 타겟 자동 인수
Alloy C ─▶ 정상 운영 중
```

1. 클러스터링이 장애 감지 (gossip 프로토콜)
2. 나머지 인스턴스가 장애 Pod의 타겟 재분배
3. 장애 Pod 복구 후 다시 재분배

### 백엔드 장애

WAL 덕분에 백엔드(Mimir/Loki) 장애 시에도 데이터를 로컬에 버퍼링합니다.
`max_keepalive_time` 기간 내 복구 시 데이터 유실 없이 재전송합니다.

---

## 다음 단계

- [Alloy 자체 모니터링](./monitoring-guide.md)
- [트러블슈팅 가이드](./troubleshooting-guide.md)
