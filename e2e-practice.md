# End-to-End 실습: 메트릭 + 로그 + 트레이스 통합 수집

이 실습에서는 Alloy를 사용하여 샘플 앱의 메트릭, 로그, 트레이스를 모두 수집하고
Grafana에서 확인하는 전체 파이프라인을 구성합니다.

---

## 실습 목표

```
샘플 앱 (nginx + OTel)
      │
      ├── 메트릭 ──▶ Alloy (prometheus.scrape) ──▶ Mimir ──▶ Grafana
      ├── 로그   ──▶ Alloy (loki.source.kubernetes) ──▶ Loki ──▶ Grafana
      └── 트레이스 ──▶ Alloy (otelcol.receiver.otlp) ──▶ Tempo ──▶ Grafana
```

---

## 사전 확인

```bash
# 백엔드 서비스 확인
kubectl get svc -n monitoring
# mimir-nginx, loki-gateway, tempo 서비스가 있어야 함

# Alloy 설치 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy

# Alloy Web UI 접속
kubectl port-forward svc/alloy 12345:12345 -n monitoring &
curl http://localhost:12345/ready
# {"status":"ready"}
```

---

## Step 1: 샘플 앱 배포

메트릭 어노테이션이 달린 nginx 앱을 배포합니다.

```yaml
# app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9113"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
        - name: nginx-exporter
          image: nginx/nginx-prometheus-exporter:1.1
          args:
            - -nginx.scrape-uri=http://localhost/stub_status
          ports:
            - containerPort: 9113
```

```yaml
# app/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: default
spec:
  selector:
    app: sample-app
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: metrics
      port: 9113
      targetPort: 9113
```

```bash
kubectl apply -f app/deployment.yaml
kubectl apply -f app/service.yaml

# 배포 확인
kubectl get pods -l app=sample-app
```

---

## Step 2: Alloy 설정 구성

`helm/values.yaml`을 수정하여 메트릭 + 로그 + 트레이스를 모두 수집합니다.

```bash
cp helm/values.yaml my-values.yaml
```

`my-values.yaml`의 `alloy.configMap.content`를 아래와 같이 설정:

```alloy
// ============================================
// 1. 로깅 설정
// ============================================
logging {
  level  = "info"
  format = "logfmt"
}

// ============================================
// 2. 메트릭 파이프라인
// ============================================
discovery.kubernetes "pods" {
  role = "pod"
}

discovery.relabel "scrape_targets" {
  targets = discovery.kubernetes.pods.targets

  // prometheus.io/scrape=true 어노테이션 있는 Pod만
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_scrape"]
    regex         = "true"
    action        = "keep"
  }

  // Running 상태만
  rule {
    source_labels = ["__meta_kubernetes_pod_phase"]
    regex         = "Running"
    action        = "keep"
  }

  // 포트 어노테이션 적용
  rule {
    source_labels = ["__address__", "__meta_kubernetes_pod_annotation_prometheus_io_port"]
    regex         = "([^:]+)(?::\\d+)?;(\\d+)"
    replacement   = "$1:$2"
    target_label  = "__address__"
  }

  // 메트릭 경로 어노테이션 적용
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_path"]
    regex         = "(.+)"
    target_label  = "__metrics_path__"
  }

  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "app"
  }
}

prometheus.scrape "pods" {
  targets         = discovery.relabel.scrape_targets.output
  forward_to      = [prometheus.remote_write.mimir.receiver]
  scrape_interval = "30s"
  job_name        = "kubernetes-pods"
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir-nginx.monitoring.svc.cluster.local:8080/api/v1/push"
    headers = {
      "X-Scope-OrgID" = "demo",
    }
  }
  external_labels = {
    cluster = "eks-demo",
  }
}

// ============================================
// 3. 로그 파이프라인
// ============================================
loki.source.kubernetes "pods" {
  targets    = discovery.relabel.scrape_targets.output
  forward_to = [loki.write.loki.receiver]
}

loki.write "loki" {
  endpoint {
    url = "http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push"
    headers = {
      "X-Scope-OrgID" = "demo",
    }
  }
  external_labels = {
    cluster = "eks-demo",
  }
}

// ============================================
// 4. 트레이스 파이프라인
// ============================================
otelcol.receiver.otlp "default" {
  grpc {
    endpoint = "0.0.0.0:4317"
  }
  http {
    endpoint = "0.0.0.0:4318"
  }
  output {
    traces = [otelcol.processor.batch.default.input]
  }
}

otelcol.processor.batch "default" {
  timeout         = "5s"
  send_batch_size = 1000
  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}

otelcol.exporter.otlp "tempo" {
  client {
    endpoint = "tempo.monitoring.svc.cluster.local:4317"
    tls {
      insecure = true
    }
    headers = {
      "X-Scope-OrgID" = "demo",
    }
  }
}

// ============================================
// 5. Alloy 자체 모니터링
// ============================================
prometheus.scrape "alloy_self" {
  targets = [{
    "__address__" = "localhost:12345",
    "job"         = "alloy",
  }]
  forward_to      = [prometheus.remote_write.mimir.receiver]
  scrape_interval = "15s"
}
```

---

## Step 3: Alloy 설치 / 재적용

```bash
# 신규 설치
helm install alloy grafana/alloy \
  --namespace monitoring \
  --values my-values.yaml \
  --version 0.9.0

# 또는 기존 설치 업그레이드
helm upgrade alloy grafana/alloy \
  --namespace monitoring \
  --values my-values.yaml

# Pod 재시작 확인
kubectl rollout status daemonset/alloy -n monitoring
```

---

## Step 4: 파이프라인 동작 확인

### Web UI 확인

```bash
kubectl port-forward svc/alloy 12345:12345 -n monitoring &
```

1. `http://localhost:12345` → **Graph** 탭 → 파이프라인 연결 확인
2. **Components** 탭 → `prometheus.scrape.pods` 클릭 → 타겟 수 확인

### 메트릭 수집 확인

```bash
# 수집 중인 타겟 확인
curl -s http://localhost:12345/api/v0/component/discovery.relabel.scrape_targets \
  | python3 -m json.tool | grep -c "__address__"

# Remote Write 성공 확인
curl -s http://localhost:12345/metrics \
  | grep prometheus_remote_storage_succeeded_samples_total
```

### 로그 수집 확인

```bash
# 드롭된 로그 없는지 확인
curl -s http://localhost:12345/metrics | grep loki_process_dropped_lines_total

# Loki에서 직접 확인 (Loki port-forward 후)
kubectl port-forward svc/loki-gateway 3100:80 -n monitoring &
curl "http://localhost:3100/loki/api/v1/query?query={app=\"sample-app\"}" \
  -H "X-Scope-OrgID: demo"
```

### 트레이스 수신 확인

```bash
# OTLP 포트 리스닝 확인
kubectl exec -n monitoring ds/alloy -- ss -tlnp | grep 4317

# span 수신 메트릭 확인
curl -s http://localhost:12345/metrics | grep otelcol_receiver_accepted_spans
```

---

## Step 5: Grafana에서 확인

### 메트릭 확인

1. Grafana → Explore → Mimir 데이터소스 선택
2. `X-Scope-OrgID: demo` 헤더 설정
3. 쿼리 입력:

```promql
# sample-app 메트릭 확인
nginx_connections_active{app="sample-app"}

# Alloy 자체 메트릭
alloy_build_info
```

### 로그 확인

1. Grafana → Explore → Loki 데이터소스 선택
2. `X-Scope-OrgID: demo` 헤더 설정
3. 쿼리 입력:

```logql
{app="sample-app", namespace="default"}
```

### 트레이스 확인

1. Grafana → Explore → Tempo 데이터소스 선택
2. `X-Scope-OrgID: demo` 헤더 설정
3. TraceQL로 검색:

```
{resource.service.name="sample-app"}
```

---

## Step 6: 트러블슈팅 연습

### 의도적 오류 생성

```bash
# 1. 존재하지 않는 Mimir 엔드포인트로 변경 후 → Remote Write 실패 알림 확인
# 2. RBAC 제거 후 → 타겟 발견 실패 확인
# 3. Pod 어노테이션 제거 후 → 특정 타겟 수집 중단 확인
```

### 복구 및 Hot Reload

```bash
# 설정 수정 후 Pod 재시작 없이 반영
curl -X POST http://localhost:12345/-/reload
```

---

## 실습 완료 후 정리

```bash
kubectl delete -f app/deployment.yaml
kubectl delete -f app/service.yaml
helm uninstall alloy -n monitoring
```

---

## 학습 요약

| 기능 | Alloy 컴포넌트 | 백엔드 |
|---|---|---|
| 메트릭 수집 | `discovery.kubernetes` + `prometheus.scrape` + `prometheus.remote_write` | Mimir |
| 로그 수집 | `loki.source.kubernetes` + `loki.write` | Loki |
| 트레이스 수집 | `otelcol.receiver.otlp` + `otelcol.processor.batch` + `otelcol.exporter.otlp` | Tempo |
| 자체 모니터링 | `prometheus.scrape` (localhost:12345) | Mimir |
