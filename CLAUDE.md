# alloy-practice — 프로젝트 가이드

## 프로젝트 설정
- 환경: EKS
- Alloy 버전: 1.x (alloy Helm chart)
- 네임스페이스: monitoring
- 백엔드: Mimir (메트릭), Loki (로그), Tempo (트레이스)
- 앱 이름: alloy

---

## 디렉토리 구조

```
alloy-practice/
├── CLAUDE.md                  # 이 파일 (자동 로드)
├── .claude/
│   ├── settings.json
│   └── commands/              # /new-doc, /new-runbook, /review-doc, /add-troubleshooting, /search-kb
├── agents/                    # doc-writer, pipeline-designer, integration-advisor, troubleshooter
├── templates/                 # service-doc, runbook, incident-report
├── rules/                     # doc-writing, alloy-conventions, security-checklist, monitoring
├── config/                    # River 설정 파일
├── helm/                      # Helm values 파일
└── *-guide.md                 # 주제별 가이드 문서
```

---

## 커스텀 슬래시 명령어

| 명령어 | 설명 | 사용 예시 |
|--------|------|---------|
| `/new-doc` | 새 가이드 문서 생성 | `/new-doc pyroscope-profiling` |
| `/new-runbook` | 새 런북 생성 | `/new-runbook Alloy 컴포넌트 장애 대응` |
| `/review-doc` | 문서 검토 | `/review-doc config-language-guide.md` |
| `/add-troubleshooting` | 트러블슈팅 케이스 추가 | `/add-troubleshooting 메트릭 수집 누락` |
| `/search-kb` | 지식베이스 검색 | `/search-kb River 컴포넌트 참조` |

---

## 가이드 문서 목록

| 문서 | 주제 |
|------|------|
| `install.md` | Alloy 설치 (Helm + EKS) |
| `architecture-guide.md` | Alloy 아키텍처 |
| `config-language-guide.md` | River 설정 언어 |
| `metrics-guide.md` | 메트릭 수집 파이프라인 |
| `logs-guide.md` | 로그 수집 파이프라인 |
| `traces-guide.md` | 트레이스 수집 파이프라인 |
| `otel-guide.md` | OTel 컴포넌트 연동 |
| `kubernetes-discovery-guide.md` | K8s 자동 검색 |
| `clustering-guide.md` | Alloy 클러스터 모드 |
| `monitoring-guide.md` | Alloy 자체 모니터링 |
| `troubleshooting-guide.md` | 트러블슈팅 |
| `e2e-practice.md` | 엔드투엔드 실습 |

---

## 핵심 명령어

```bash
# Alloy 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy

# Alloy UI 접속 (포트포워딩)
kubectl port-forward -n monitoring svc/alloy 12345:12345

# Alloy 내부 메트릭
curl http://localhost:12345/metrics

# River 설정 검증
alloy fmt config.alloy
alloy run --stability.level=generally-available config.alloy
```
