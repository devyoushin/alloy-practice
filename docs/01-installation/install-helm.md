# Alloy 설치 (Helm)

EKS 환경에 Grafana Alloy를 Helm으로 설치한다.

## 사전 준비

```bash
kubectl version --client
helm version
kubectl get nodes
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
```

## values 준비

```bash
cp ../../ops/config/helm/values.yaml my-values.yaml
```

## 설치

```bash
helm install alloy grafana/alloy \
  --namespace monitoring \
  --values my-values.yaml \
  --version 0.9.0
```

## 확인

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
kubectl get daemonset -n monitoring
kubectl port-forward svc/alloy 12345:12345 -n monitoring
curl http://localhost:12345/ready
```

## 업그레이드 / 삭제

```bash
helm upgrade alloy grafana/alloy -n monitoring --values my-values.yaml
helm uninstall alloy -n monitoring
```
