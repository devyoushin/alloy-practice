# 보안 체크리스트 — alloy-practice

## Grafana Alloy 보안 설정

### 자격증명 관리
- [ ] 백엔드 자격증명 환경변수 또는 K8s Secret 참조 (하드코딩 금지)
- [ ] River 파일에 비밀번호, 토큰 직접 입력 금지
- [ ] K8s Secret을 환경변수로 마운트하여 `env()` 참조

### 네트워크 보안
- [ ] Alloy UI 포트(12345) 내부망 제한 (외부 노출 금지)
- [ ] Alloy → 백엔드 TLS 설정
- [ ] OTLP 수신 포트(4317/4318) 내부망 제한
- [ ] NetworkPolicy로 Alloy 접근 제한

### 데이터 보안
- [ ] 민감 레이블 relabel로 제거 (password, token, user_id)
- [ ] 로그 내 민감 데이터 마스킹 (loki.process regex replace)
- [ ] Span attribute 필터링 (otelcol.processor.attributes)

### RBAC 설정
- [ ] K8s ServiceAccount 최소 권한 (pods, services list/watch만 허용)
- [ ] ClusterRole 필요 최소 리소스만 허용

## 정기 보안 점검 (월별)
- [ ] River 설정 파일 자격증명 스캔 (truffleHog 등)
- [ ] 백엔드 자격증명 순환
- [ ] Alloy 버전 업데이트 확인
- [ ] 수집 중인 민감 데이터 감사
