# Grafana Alloy 설치 가이드

EKS 환경에 Helm으로 Grafana Alloy를 설치합니다.

---

## 사전 요구사항

```bash
kubectl version --client   # >= 1.25
helm version               # >= 3.10
kubectl get nodes          # EKS 접속 확인
```

---

## 1. Helm Repository 추가

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 버전 확인
helm search repo grafana/alloy --versions | head -5
```

---

## 2. 네임스페이스 생성

```bash
kubectl create namespace monitoring
```

---

## 3. Helm Values 준비

`../ops/config/helm/values.yaml`을 복사하여 환경에 맞게 수정합니다.

```bash
cp ../ops/config/helm/values.yaml my-values.yaml
```

수정 필수 항목:
- `alloy.configMap.content` 내 Mimir/Loki/Tempo 엔드포인트
- 백엔드가 같은 클러스터에 있다면 서비스 이름 사용 가능

---

## 4. Alloy 설치

```bash
helm install alloy grafana/alloy \
  --namespace monitoring \
  --values my-values.yaml \
  --version 0.9.0
```

설치 확인:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
kubectl get daemonset -n monitoring
kubectl get configmap -n monitoring alloy
```

---

## 5. 포트 포워딩 & UI 접근

Alloy는 기본적으로 `:12345` 포트에 Web UI를 제공합니다.

```bash
kubectl port-forward svc/alloy 12345:12345 -n monitoring &
```

브라우저에서 `http://localhost:12345` 접속:
- **Graph** 탭: 파이프라인 컴포넌트 연결 시각화
- **Components** 탭: 각 컴포넌트 상태 및 exports 확인
- **Debugging** 탭: 실시간 로그 확인

---

## 6. 헬스 체크

```bash
# Ready 상태 확인
curl http://localhost:12345/ready

# Metrics 확인 (Alloy 자체 메트릭)
curl http://localhost:12345/metrics | grep alloy_build_info
```

---

## 7. 설정 확인 및 재로드

```bash
# 현재 설정 확인
kubectl get configmap alloy -n monitoring -o yaml

# 설정 변경 후 재로드 (pod 재시작 없이)
curl -X POST http://localhost:12345/-/reload
```

---

## 8. 업그레이드 및 제거

```bash
# 업그레이드
helm upgrade alloy grafana/alloy \
  --namespace monitoring \
  --values my-values.yaml

# 제거
helm uninstall alloy -n monitoring
```

---

## 트러블슈팅

### Pod가 CrashLoopBackOff 상태인 경우

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --previous

# 설정 파일 문법 오류 확인
kubectl describe pod -n monitoring -l app.kubernetes.io/name=alloy
```

### 설정이 적용되지 않는 경우

```bash
# ConfigMap 내용 확인
kubectl get configmap alloy -n monitoring -o jsonpath='{.data.config\.alloy}'

# Pod 재시작
kubectl rollout restart daemonset/alloy -n monitoring
```

---

## 다음 단계

- [아키텍처 개요](./architecture-guide.md)
- [Alloy 설정 언어](./config-language-guide.md)
