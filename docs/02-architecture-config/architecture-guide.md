# Grafana Alloy 아키텍처 가이드

---

## Alloy란?

Grafana Alloy는 Grafana Labs에서 만든 오픈소스 **텔레메트리 파이프라인 수집기**입니다.
기존 Grafana Agent (Flow 모드)의 후속 버전으로, OpenTelemetry Collector를 완전히 내장하고 있으며 메트릭, 로그, 트레이스, 프로파일을 하나의 에이전트에서 처리합니다.

---

## 기존 도구와 비교

| 도구 | 메트릭 | 로그 | 트레이스 | 프로파일 | 특징 |
|---|---|---|---|---|---|
| **Prometheus** | O | X | X | X | Pull 기반 메트릭 전용 |
| **Promtail** | X | O | X | X | Loki 전용 로그 에이전트 |
| **OTel Collector** | O | O | O | X | OpenTelemetry 표준 |
| **Grafana Agent** | O | O | O | O | Alloy의 전신 (deprecated) |
| **Grafana Alloy** | O | O | O | O | 파이프라인 기반, 통합 에이전트 |

Alloy의 핵심 장점:
- **단일 에이전트**로 모든 신호 유형 처리
- **파이프라인 기반** 설정 → 컴포넌트를 레고처럼 조립
- **OTel Collector 내장** → OTLP 수신/발신 네이티브 지원
- **실시간 디버깅 UI** → 파이프라인 상태를 웹 브라우저로 확인
- **Hot reload** → Pod 재시작 없이 설정 변경 적용

---

## 핵심 아키텍처: 파이프라인 모델

Alloy는 **컴포넌트(Component)** 를 연결해 데이터 파이프라인을 구성합니다.
각 컴포넌트는 독립적으로 동작하며, exports를 통해 다른 컴포넌트에 데이터를 전달합니다.

```
[Source 컴포넌트] ──exports──▶ [Processor 컴포넌트] ──exports──▶ [Exporter 컴포넌트]
```

### 예시: 메트릭 파이프라인

```
discovery.kubernetes "pods" {
  role = "pod"
}
         │ (targets)
         ▼
prometheus.scrape "default" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [prometheus.remote_write.mimir.receiver]
}
         │ (metrics)
         ▼
prometheus.remote_write "mimir" {
  endpoint { url = "http://mimir:8080/api/v1/push" }
}
```

---

## 컴포넌트 네임스페이스

컴포넌트 이름은 `네임스페이스.유형 "레이블"` 형태입니다.

| 네임스페이스 | 역할 | 주요 컴포넌트 |
|---|---|---|
| `discovery` | 스크랩 대상 발견 | `discovery.kubernetes`, `discovery.relabel`, `discovery.file` |
| `prometheus` | 메트릭 수집/전송 | `prometheus.scrape`, `prometheus.remote_write`, `prometheus.operator.*` |
| `loki` | 로그 수집/전송 | `loki.source.kubernetes`, `loki.source.file`, `loki.write`, `loki.relabel` |
| `otelcol` | OTel 파이프라인 | `otelcol.receiver.otlp`, `otelcol.processor.batch`, `otelcol.exporter.otlp` |
| `pyroscope` | 프로파일 수집/전송 | `pyroscope.scrape`, `pyroscope.write` |
| `alloy` | Alloy 내부 설정 | `alloy.clustering` |
| `remote` | 원격 설정 로드 | `remote.kubernetes.configmap`, `remote.http` |

---

## 배포 모드

### DaemonSet (권장 — 로그 수집)

각 노드에 1개 Pod가 배포됩니다.
노드의 컨테이너 로그(`/var/log/pods`)에 직접 접근해야 하는 로그 수집에 적합합니다.

```
Node 1: [Alloy Pod] ─▶ /var/log/pods/*.log ─▶ Loki
Node 2: [Alloy Pod] ─▶ /var/log/pods/*.log ─▶ Loki
Node 3: [Alloy Pod] ─▶ /var/log/pods/*.log ─▶ Loki
```

### Deployment (메트릭 수집)

여러 replica로 배포됩니다.
클러스터링 활성화 시 스크랩 타겟을 자동으로 분산 처리합니다.

```
[Alloy Pod 1] ─▶ (타겟 1~50 담당)  ─▶ Mimir
[Alloy Pod 2] ─▶ (타겟 51~100 담당) ─▶ Mimir
[Alloy Pod 3] ─▶ (타겟 101~150 담당) ─▶ Mimir
```

### StatefulSet (클러스터링 없이 WAL 사용)

WAL(Write-Ahead Log)을 통해 재시작 후에도 데이터 유실 없이 재전송합니다.

---

## 데이터 흐름 전체 그림

```
┌──────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                           │
│                                                                  │
│  ┌────────────┐     ┌─────────────────────────────────────────┐  │
│  │ App Pods   │────▶│         Grafana Alloy                   │  │
│  │ (OTLP Push)│     │                                         │  │
│  └────────────┘     │  discovery.kubernetes                   │  │
│                     │       │                                  │  │
│  ┌────────────┐     │       ▼                                  │  │
│  │ Node Logs  │────▶│  prometheus.scrape ──▶ remote_write ────┼──┼──▶ Mimir
│  │ /var/log/  │     │                                         │  │
│  └────────────┘     │  loki.source.kubernetes ──▶ loki.write ─┼──┼──▶ Loki
│                     │                                         │  │
│  ┌────────────┐     │  otelcol.receiver.otlp                  │  │
│  │ PodMonitor │────▶│       │                                  │  │
│  │ ServiceMon │     │       ▼                                  │  │
│  └────────────┘     │  otelcol.exporter.otlp ─────────────────┼──┼──▶ Tempo
│                     └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## RBAC 권한

Alloy가 Kubernetes API에 접근하기 위해 필요한 권한:

```yaml
rules:
  # 서비스 디스커버리용
  - apiGroups: [""]
    resources: ["nodes", "pods", "services", "endpoints", "namespaces"]
    verbs: ["get", "list", "watch"]
  # 로그 수집용 (노드의 kubelet 접근)
  - apiGroups: [""]
    resources: ["nodes/proxy"]
    verbs: ["get"]
  # PodMonitor / ServiceMonitor 지원
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["podmonitors", "servicemonitors", "probes"]
    verbs: ["get", "list", "watch"]
  # 클러스터 수준 리소스
  - apiGroups: ["apps"]
    resources: ["daemonsets", "deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch"]
```

Helm chart를 사용하면 위 RBAC 리소스가 자동으로 생성됩니다.

---

## 다음 단계

- [Alloy 설정 언어 (River)](./config-language-guide.md)
- [Kubernetes 서비스 디스커버리](./kubernetes-discovery-guide.md)
