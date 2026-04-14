# Grafana Alloy on EKS — 실습 저장소

EKS 환경에서 Grafana Alloy를 처음부터 운영 수준까지 학습하는 실습 저장소입니다.

---

## 환경 정보

| 항목 | 값 |
|---|---|
| 플랫폼 | AWS EKS |
| Alloy Helm Chart | `grafana/alloy` (권장 버전: 0.9.x) |
| 네임스페이스 | `monitoring` |
| 메트릭 백엔드 | Grafana Mimir |
| 로그 백엔드 | Grafana Loki |
| 트레이스 백엔드 | Grafana Tempo |
| 리전 | `ap-northeast-2` |

---

## 사전 요구사항

```bash
# 필요 도구 확인
kubectl version --client      # >= 1.25
helm version                  # >= 3.10
aws --version                 # AWS CLI v2

# EKS 클러스터 접속 확인
kubectl get nodes

# 백엔드 서비스 확인 (선택)
kubectl get svc -n monitoring  # Mimir, Loki, Tempo
```

---

## 빠른 시작 (Quick Start)

```bash
# 1. Helm repo 추가
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 2. 네임스페이스 생성
kubectl create namespace monitoring

# 3. values 파일 수정
cp helm/values.yaml my-values.yaml
# my-values.yaml에서 Mimir/Loki/Tempo 엔드포인트 수정

# 4. Alloy 설치
helm install alloy grafana/alloy \
  --namespace monitoring \
  --values my-values.yaml \
  --version 0.9.0

# 5. 헬스 체크
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
kubectl port-forward svc/alloy 12345:12345 -n monitoring &
curl http://localhost:12345/ready
```

---

## 학습 경로

### 1단계: 설치
- [Helm으로 Alloy 설치](./install.md)

### 2단계: 핵심 개념
- [아키텍처 개요](./architecture-guide.md)
- [Alloy 설정 언어 (River)](./config-language-guide.md)
- [Kubernetes 서비스 디스커버리](./kubernetes-discovery-guide.md)

### 3단계: 데이터 파이프라인
- [메트릭 수집 — Prometheus → Mimir](./metrics-guide.md)
- [로그 수집 — Kubernetes → Loki](./logs-guide.md)
- [트레이스 수집 — OTel → Tempo](./traces-guide.md)

### 4단계: OpenTelemetry 통합
- [OpenTelemetry 파이프라인](./otel-guide.md)

### 5단계: 운영 심화
- [클러스터링 & 고가용성](./clustering-guide.md)
- [Alloy 자체 모니터링](./monitoring-guide.md)

### 6단계: 문제 해결
- [트러블슈팅 가이드](./troubleshooting-guide.md)

### 실습
- [End-to-End 실습 (메트릭 + 로그 + 트레이스)](./e2e-practice.md)

---

## 저장소 구조

```
alloy-practice/
├── README.md
├── CLAUDE.md
├── install.md                        # Helm 설치 가이드
├── architecture-guide.md             # 아키텍처 설명
├── config-language-guide.md          # Alloy 설정 언어 (River)
├── kubernetes-discovery-guide.md     # Kubernetes 서비스 디스커버리
├── metrics-guide.md                  # 메트릭 수집 파이프라인
├── logs-guide.md                     # 로그 수집 파이프라인
├── traces-guide.md                   # 트레이스 수집 파이프라인
├── otel-guide.md                     # OpenTelemetry 통합
├── clustering-guide.md               # 클러스터링 & HA
├── monitoring-guide.md               # 자체 모니터링
├── troubleshooting-guide.md          # 트러블슈팅
├── e2e-practice.md                   # End-to-End 실습
├── helm/
│   ├── values.yaml                   # 개발/테스트용 Helm values
│   └── values-ha.yaml                # 운영 HA Helm values
└── config/
    ├── metrics.alloy                 # 메트릭 전용 설정
    ├── logs.alloy                    # 로그 전용 설정
    ├── traces.alloy                  # 트레이스 전용 설정
    └── full-stack.alloy              # 메트릭 + 로그 + 트레이스 통합 설정
```

---

## 아키텍처 요약

```
┌─────────────────────────────────────────────────────────┐
│                    Grafana Alloy (DaemonSet/Deployment)  │
│                                                         │
│  discovery.kubernetes ──▶ prometheus.scrape             │
│                               │                         │
│                               ▼                         │
│                       prometheus.remote_write ──────────┼──▶ Mimir
│                                                         │
│  loki.source.kubernetes ──▶ loki.write ─────────────────┼──▶ Loki
│                                                         │
│  otelcol.receiver.otlp ──▶ otelcol.processor.batch     │
│                               │                         │
│                               ▼                         │
│                       otelcol.exporter.otlp ────────────┼──▶ Tempo
└─────────────────────────────────────────────────────────┘
```

| 컴포넌트 유형 | 예시 | 역할 |
|---|---|---|
| **discovery** | `discovery.kubernetes` | 스크랩 대상 동적 발견 |
| **prometheus** | `prometheus.scrape`, `prometheus.remote_write` | 메트릭 수집 및 전송 |
| **loki** | `loki.source.kubernetes`, `loki.write` | 로그 수집 및 전송 |
| **otelcol** | `otelcol.receiver.otlp`, `otelcol.exporter.otlp` | OTel 파이프라인 |
| **pyroscope** | `pyroscope.scrape`, `pyroscope.write` | 프로파일링 수집 및 전송 |

---

## 참고 링크

- [Grafana Alloy 공식 문서](https://grafana.com/docs/alloy/latest/)
- [Alloy Helm Chart](https://github.com/grafana/alloy/tree/main/operations/helm/charts/alloy)
- [Alloy 컴포넌트 레퍼런스](https://grafana.com/docs/alloy/latest/reference/components/)
- [Alloy 설정 언어 문법](https://grafana.com/docs/alloy/latest/get-started/configuration-syntax/)
