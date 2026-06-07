# Alloy 업그레이드 가이드

Alloy 업그레이드는 Helm chart, Alloy image, River 설정 문법, component 안정성에 영향을 줄 수 있습니다. 업그레이드 전 설정 검증과 backend 연결 상태를 확인합니다.

## 1. 사전 점검

```bash
export NAMESPACE="monitoring"
export RELEASE="alloy"
export CHART_VERSION="0.10.0"
export VALUES_FILE="my-values.yaml"

helm status ${RELEASE} -n ${NAMESPACE}
helm history ${RELEASE} -n ${NAMESPACE}
helm get values ${RELEASE} -n ${NAMESPACE} > values-before-upgrade.yaml
kubectl get pods,daemonset,svc -n ${NAMESPACE} -l app.kubernetes.io/name=alloy
```

River 설정을 별도로 보관합니다.

```bash
kubectl get configmap -n ${NAMESPACE} -l app.kubernetes.io/name=alloy -o yaml > alloy-config-before-upgrade.yaml
```

## 2. Helm 업그레이드

```bash
helm repo update grafana
helm upgrade ${RELEASE} grafana/alloy \
  --namespace ${NAMESPACE} \
  --values ${VALUES_FILE} \
  --version ${CHART_VERSION} \
  --timeout 10m \
  --wait
```

## 3. 확인

```bash
kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/name=alloy
kubectl rollout status daemonset/${RELEASE} -n ${NAMESPACE}
kubectl port-forward svc/${RELEASE} 12345:12345 -n ${NAMESPACE}
curl http://localhost:12345/ready
```

Alloy UI의 component graph, Prometheus remote_write, Loki write, OTLP exporter 상태를 확인합니다.

## 4. 롤백

```bash
helm history ${RELEASE} -n ${NAMESPACE}
helm rollback ${RELEASE} <REVISION> -n ${NAMESPACE} --wait
```

새 River 문법이나 component 옵션을 사용했다면 rollback 전에 이전 설정으로 되돌려야 합니다.

## 5. systemd / Docker Compose

systemd 설치는 Alloy 바이너리와 River 설정을 백업한 뒤 서비스를 재시작합니다. Docker Compose 설치는 image tag를 변경하고 `docker compose pull && docker compose up -d`를 실행합니다. 모든 방식에서 `/ready`와 backend 전송 metric을 확인합니다.

