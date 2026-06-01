# Alloy Docs

Alloy를 처음 보는 사람이 설치부터 River 설정과 파이프라인 운영까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `architecture-guide.md` | Alloy 배포 구조와 데이터 흐름 |
| 3 | `config-language-guide.md` | River 설정 언어 |
| 4 | `kubernetes-discovery-guide.md` | Kubernetes service discovery |
| 5 | `metrics-guide.md`, `logs-guide.md`, `traces-guide.md` | 각 signal 파이프라인 |
| 6 | `otel-guide.md` | OpenTelemetry 컴포넌트 통합 |
| 7 | `clustering-guide.md`, `monitoring-guide.md` | 클러스터링과 자체 모니터링 |
| 8 | `rules/README.md` | 문서와 운영 규칙 |
| 9 | `agents/README.md` | AI 작업 지침 |
| 10 | `templates/README.md` | 문서 템플릿 |
| 11 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/기초 | `install.md`, `architecture-guide.md`, `config-language-guide.md` |
| Discovery/Pipeline | `kubernetes-discovery-guide.md`, `metrics-guide.md`, `logs-guide.md`, `traces-guide.md` |
| 통합 | `otel-guide.md` |
| 운영 | `clustering-guide.md`, `monitoring-guide.md`, `e2e-practice.md`, `troubleshooting-guide.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install.md`
2. `architecture-guide.md`
3. `config-language-guide.md`
4. `kubernetes-discovery-guide.md`
5. `metrics-guide.md`
6. `logs-guide.md`
7. `traces-guide.md`
8. `otel-guide.md`
9. `clustering-guide.md`
10. `monitoring-guide.md`
11. `rules/README.md`
12. `agents/README.md`
13. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Alloy Helm values와 River 설정 예시
