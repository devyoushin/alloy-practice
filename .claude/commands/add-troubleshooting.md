Alloy 트러블슈팅 케이스를 추가합니다.

**사용법**: `/add-troubleshooting <증상 설명>`  **예시**: `/add-troubleshooting 메트릭 전송 실패`

형식:
```markdown
### <증상>
**원인**: <근본 원인>
**확인**:
\`\`\`bash
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy
curl http://alloy:12345/metrics | grep alloy_
# Alloy UI: http://localhost:12345 (포트포워딩)
\`\`\`
**해결**: <해결 방법>
```
`troubleshooting-guide.md`에 추가하세요.
