# [알림명] — Grafana Alloy 런북

> 심각도: Critical / Warning / Info
> 담당팀: | 최종 수정: YYYY-MM-DD

## 알림 요약
<!-- 이 알림이 무엇을 감지하는지 한 문장으로 -->

## 알림 조건
<!-- 알림 트리거 조건 설명 -->

## 영향도
<!-- 이 알림이 발생했을 때 사용자/서비스에 미치는 영향 -->

## 즉시 확인 사항

```bash
# 1. Alloy Pod 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy

# 2. Alloy 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --tail=50

# 3. Alloy UI 컴포넌트 상태 확인
# http://<alloy-pod-ip>:12345/graph
```

## 진단 단계

### 1단계: 컴포넌트 상태 확인
```bash
# Alloy 내부 메트릭 확인
curl http://alloy:12345/metrics | grep alloy_
```

### 2단계: 원인 분석
<!-- 일반적인 원인 목록 -->

### 3단계: 해결 조치
<!-- 단계별 해결 방법 -->

## 에스컬레이션
- 15분 내 해결 불가 시: 팀 리드 호출
- 전체 수집 중단 시: 인시던트 선언

## 참고
- 관련 Grafana 대시보드:
- 관련 문서:
