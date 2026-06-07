# Alloy Docs

Alloy를 처음 보는 사람이 설치부터 River 설정과 파이프라인 운영까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install/install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `install/upgrade/` | Alloy 업그레이드 |
| 3 | `architecture-guide.md` | Alloy 배포 구조와 데이터 흐름 |
| 4 | `config-language-guide.md` | River 설정 언어 |
| 5 | `kubernetes-discovery-guide.md` | Kubernetes service discovery |
| 6 | `metrics-guide.md`, `logs-guide.md`, `traces-guide.md` | 각 signal 파이프라인 |
| 7 | `otel-guide.md` | OpenTelemetry 컴포넌트 통합 |
| 8 | `clustering-guide.md`, `monitoring-guide.md` | 클러스터링과 자체 모니터링 |
| 9 | `rules/README.md` | 문서와 운영 규칙 |
| 10 | `agents/README.md` | AI 작업 지침 |
| 11 | `templates/README.md` | 문서 템플릿 |
| 12 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/기초 | `install/install.md`, `install/upgrade/`, `architecture-guide.md`, `config-language-guide.md` |
| Discovery/Pipeline | `kubernetes-discovery-guide.md`, `metrics-guide.md`, `logs-guide.md`, `traces-guide.md` |
| 통합 | `otel-guide.md` |
| 운영 | `clustering-guide.md`, `monitoring-guide.md`, `e2e-practice.md`, `troubleshooting-guide.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install/install.md`
2. `install/upgrade/`
3. `architecture-guide.md`
4. `config-language-guide.md`
5. `kubernetes-discovery-guide.md`
6. `metrics-guide.md`
7. `logs-guide.md`
8. `traces-guide.md`
9. `otel-guide.md`
10. `clustering-guide.md`
11. `monitoring-guide.md`
12. `rules/README.md`
13. `agents/README.md`
14. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Alloy Helm values와 River 설정 예시
