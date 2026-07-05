# 트러블슈팅 가이드

---

## 진단 명령어 모음

```bash
# Pod 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy

# 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --tail=100

# 이전 컨테이너 로그 (CrashLoopBackOff 시)
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --previous

# 이벤트 확인
kubectl describe pod -n monitoring -l app.kubernetes.io/name=alloy

# 설정 확인
kubectl get configmap alloy -n monitoring -o jsonpath='{.data.config\.alloy}'

# Web UI 접속
kubectl port-forward svc/alloy 12345:12345 -n monitoring
```

---

## 문제 1: Pod CrashLoopBackOff

### 증상

```bash
$ kubectl get pods -n monitoring
NAME           READY   STATUS             RESTARTS   AGE
alloy-xxxxx    0/1     CrashLoopBackOff   5          3m
```

### 원인 1: 설정 파일 문법 오류

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --previous
# 출력 예시: level=error msg="error during the initial gragent load" err="..."
```

해결:
```bash
# 로컬에서 문법 검사
docker run --rm -v $(pwd)/../ops/config/alloy:/config grafana/alloy:latest \
  alloy fmt /config/full-stack.alloy

# 또는 alloy binary 직접 사용
alloy fmt ../ops/config/alloy/full-stack.alloy
```

### 원인 2: 백엔드 엔드포인트 연결 실패

```bash
# ConfigMap의 엔드포인트 URL 확인
kubectl get configmap alloy -n monitoring -o yaml | grep -A5 url

# 서비스 존재 여부 확인
kubectl get svc -n monitoring | grep mimir
kubectl get svc -n monitoring | grep loki
```

---

## 문제 2: 메트릭이 Mimir에 수집되지 않음

### 확인 순서

```bash
# 1. Alloy Pod 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy

# 2. Web UI에서 컴포넌트 상태 확인
kubectl port-forward svc/alloy 12345:12345 -n monitoring &
curl http://localhost:12345/api/v0/component  # 컴포넌트 목록

# 3. Remote Write 상태 확인
curl http://localhost:12345/metrics | grep prometheus_remote_storage

# 4. 발견된 타겟 수 확인
curl http://localhost:12345/api/v0/component/prometheus.scrape.pods
```

### 원인 1: 타겟이 0개인 경우

```bash
# discovery.kubernetes 컴포넌트 확인
curl http://localhost:12345/api/v0/component/discovery.kubernetes.pods
# "targets": [] 이면 RBAC 문제 의심

# ServiceAccount 권한 확인
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:alloy
kubectl auth can-i list nodes --as=system:serviceaccount:monitoring:alloy
```

해결:
```bash
# ClusterRole/ClusterRoleBinding 확인
kubectl get clusterrolebinding -l app.kubernetes.io/name=alloy
kubectl describe clusterrole alloy
```

### 원인 2: 릴레이블링으로 타겟이 필터링됨

```alloy
// discovery.relabel에서 drop action이 너무 강하게 걸려있는지 확인
// Web UI → Components → discovery.relabel.pods → 출력 타겟 수 확인
```

### 원인 3: Remote Write 실패

```bash
# Remote Write 에러 확인
curl http://localhost:12345/metrics | grep -E "failed_samples|send_failed"

# Mimir 접근 가능 여부 확인
kubectl exec -n monitoring deploy/alloy -- \
  wget -qO- http://mimir-nginx.monitoring.svc:8080/ready
```

---

## 문제 3: 로그가 Loki에 수집되지 않음

### 확인 순서

```bash
# loki.source.kubernetes 컴포넌트 확인
curl http://localhost:12345/api/v0/component/loki.source.kubernetes.pods

# 드롭된 로그 확인
curl http://localhost:12345/metrics | grep loki_process_dropped_lines_total

# Loki 접근 가능 여부 확인
kubectl exec -n monitoring ds/alloy -- \
  wget -qO- http://loki-gateway.monitoring.svc/ready
```

### 원인 1: Pod 로그에 접근 권한 없음

```bash
# DaemonSet인지 확인 (로그 수집은 DaemonSet 필요)
kubectl get daemonset -n monitoring alloy
```

### 원인 2: 위치 파일 문제

```bash
# 위치 파일 확인
kubectl exec -n monitoring ds/alloy -- \
  cat /var/lib/alloy/positions.yaml
```

---

## 문제 4: 트레이스가 Tempo에 수집되지 않음

```bash
# OTel receiver 상태 확인
curl http://localhost:12345/api/v0/component/otelcol.receiver.otlp.default

# 수신된 span 수
curl http://localhost:12345/metrics | grep otelcol_receiver_accepted_spans

# 전송 실패 span 수
curl http://localhost:12345/metrics | grep otelcol_exporter_send_failed_spans

# Tempo 접근 가능 여부 확인
kubectl exec -n monitoring deploy/alloy -- \
  wget -qO- http://tempo.monitoring.svc:3100/ready

# 4317 포트 리스닝 확인
kubectl exec -n monitoring deploy/alloy -- ss -tlnp | grep 4317
```

---

## 문제 5: 설정 변경 후 반영 안 됨

### Hot Reload 사용

```bash
# Pod 재시작 없이 설정 반영
kubectl port-forward svc/alloy 12345:12345 -n monitoring &
curl -X POST http://localhost:12345/-/reload

# 응답: {"status":"OK"}
```

### ConfigMap이 업데이트되지 않은 경우

```bash
# ConfigMap 현재 내용 확인
kubectl get configmap alloy -n monitoring -o jsonpath='{.data.config\.alloy}'

# Helm upgrade로 ConfigMap 업데이트
helm upgrade alloy grafana/alloy \
  --namespace monitoring \
  --values my-values.yaml

# Pod 재시작
kubectl rollout restart daemonset/alloy -n monitoring
# 또는
kubectl rollout restart deployment/alloy -n monitoring
```

---

## 문제 6: 클러스터링이 동작하지 않음

```bash
# 클러스터 멤버 확인
curl http://localhost:12345/api/v0/component/alloy.clustering.default

# Headless Service 확인 (클러스터링에 필수)
kubectl get svc -n monitoring alloy
# ClusterIP가 "None"이어야 함

# Pod 간 통신 확인 (12345 포트)
kubectl exec -n monitoring <alloy-pod-1> -- \
  wget -qO- http://<alloy-pod-2>:12345/ready
```

---

## 문제 7: 메모리 사용량 급증

```bash
# 메모리 사용량 확인
kubectl top pods -n monitoring -l app.kubernetes.io/name=alloy

# 수집 중인 샘플 수 확인
curl http://localhost:12345/metrics | grep prometheus_tsdb_head_samples_appended_total
```

해결 방법:
1. `scrape_interval` 늘리기 (기본 15s → 30s)
2. 불필요한 메트릭 drop 릴레이블링 추가
3. `otelcol.processor.memory_limiter` 추가
4. Helm values에서 메모리 limit 증가

---

## 유용한 쿼리 (Grafana Explore)

```promql
# Alloy 인스턴스별 스크랩 타겟 수
count by (pod) (up{job="alloy"})

# Remote Write 성공률
rate(prometheus_remote_storage_succeeded_samples_total[5m])
/ (
  rate(prometheus_remote_storage_succeeded_samples_total[5m])
  + rate(prometheus_remote_storage_failed_samples_total[5m])
)

# 전송 대기 샘플 수
prometheus_remote_storage_samples_pending

# 스크랩 평균 소요 시간
avg by (job) (scrape_duration_seconds)
```

---

## 참고 링크

- [Alloy 공식 트러블슈팅 문서](https://grafana.com/docs/alloy/latest/troubleshoot/)
- [Alloy GitHub Issues](https://github.com/grafana/alloy/issues)
