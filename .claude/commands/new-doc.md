새 Alloy 가이드 문서를 생성합니다.

**사용법**: `/new-doc <주제명>`  **예시**: `/new-doc kafka-receiver`

주제 분류: config-language, metrics, logs, traces, otel, kubernetes-discovery, clustering

`<주제명>-guide.md` 생성 시 포함 내용:
- CLAUDE.md 환경 설정 반영 (EKS, Alloy 버전)
- Alloy River 설정 예시 (.alloy 파일)
- 백엔드 연동 설정 (Mimir/Loki/Tempo)
- Alloy UI 또는 kubectl 확인 명령어
- 트러블슈팅 섹션
