# alloy-practice

EKS + Grafana Alloy 기준으로 metrics/logs/traces 파이프라인, River 설정 언어, Kubernetes discovery, OpenTelemetry 통합, 클러스터링 운영을 정리한 개인 학습 문서입니다.

## 빠른 시작

- 처음 볼 문서: `docs/01-installation/install.md`
- 설치 방식: Helm / systemd / Docker Compose
- 전체 흐름: 설치 -> 아키텍처/River -> Kubernetes discovery -> metrics/logs/traces -> OTel 통합 -> 운영
- AI 작업 지침: `CLAUDE.md`

## 구조

```text
alloy-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── README.md
│   ├── agents/
│   ├── rules/
│   ├── templates/
│   └── *.md
└── ops/
    ├── README.md
    └── config/     # Alloy Helm values와 River 설정 예시
```

## 학습 경로

| 단계 | 문서 |
|------|------|
| 설치 | `docs/01-installation/install.md` |
| 핵심 개념 | `docs/architecture-guide.md`, `docs/config-language-guide.md`, `docs/kubernetes-discovery-guide.md` |
| Pipeline | `docs/metrics-guide.md`, `docs/logs-guide.md`, `docs/traces-guide.md` |
| 통합 | `docs/otel-guide.md`, `docs/e2e-practice.md` |
| 운영 | `docs/clustering-guide.md`, `docs/monitoring-guide.md`, `docs/troubleshooting-guide.md` |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Chart | `grafana/alloy` |
| Namespace | `monitoring` |
| Metrics backend | Grafana Mimir |
| Logs backend | Grafana Loki |
| Traces backend | Grafana Tempo |
