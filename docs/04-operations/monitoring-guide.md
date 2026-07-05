# Alloy 자체 모니터링 가이드

Alloy 자체의 상태를 Prometheus/Mimir로 수집하고 Grafana로 시각화합니다.
"Alloy로 Alloy를 관찰"하는 메타 모니터링 패턴입니다.

---

## Alloy 자체 메트릭

Alloy는 `:12345/metrics` 엔드포인트에서 Prometheus 형식 메트릭을 노출합니다.

### 주요 메트릭

```
# 빌드 정보
alloy_build_info{version, goversion, os, arch}

# 컴포넌트 상태
alloy_component_controller_running_components   # 실행 중인 컴포넌트 수
alloy_component_evaluation_seconds              # 컴포넌트 평가 시간

# 메트릭 수집 상태
prometheus_scrape_samples_scraped              # 스크랩된 샘플 수
prometheus_scrape_duration_seconds             # 스크랩 소요 시간
prometheus_scrape_samples_post_metric_relabeling  # 릴레이블링 후 샘플 수

# Remote Write 상태
prometheus_remote_storage_samples_pending       # 전송 대기 중인 샘플
prometheus_remote_storage_failed_samples_total  # 전송 실패 샘플 수
prometheus_remote_storage_succeeded_samples_total  # 전송 성공 샘플 수
prometheus_remote_storage_sent_batch_duration_seconds  # 배치 전송 시간

# WAL 상태
prometheus_wal_watcher_current_segment          # 현재 WAL 세그먼트
prometheus_tsdb_wal_truncations_total           # WAL 트런케이션 횟수

# 로그 수집 상태
loki_process_dropped_lines_total                # 드롭된 로그 줄 수
loki_source_file_read_bytes_total               # 읽은 로그 바이트

# OTel 상태
otelcol_receiver_accepted_spans                 # 수신 성공 span 수
otelcol_exporter_sent_spans                     # 전송 성공 span 수
otelcol_exporter_send_failed_spans              # 전송 실패 span 수
```

---

## 자체 메트릭 수집 설정

```alloy
// Alloy 자신의 메트릭 수집
prometheus.scrape "alloy_self" {
  targets = [
    {
      "__address__" = "localhost:12345",
      "job"         = "alloy",
    },
  ]
  forward_to      = [prometheus.remote_write.mimir.receiver]
  scrape_interval = "15s"
}

prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir-nginx.monitoring.svc.cluster.local:8080/api/v1/push"
  }
  external_labels = {
    cluster   = "production-eks",
    component = "alloy",
  }
}
```

---

## 클러스터 내 모든 Alloy 인스턴스 수집

여러 Alloy Pod을 동시에 모니터링합니다.

```alloy
// Alloy Pod 발견
discovery.kubernetes "alloy_pods" {
  role = "pod"

  selectors {
    role  = "pod"
    label = "app.kubernetes.io/name=alloy"
  }
}

discovery.relabel "alloy_pods" {
  targets = discovery.kubernetes.alloy_pods.targets

  // 메트릭 포트 설정
  rule {
    source_labels = ["__meta_kubernetes_pod_container_port_number"]
    regex         = "12345"
    action        = "keep"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }

  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }

  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }
}

prometheus.scrape "alloy_fleet" {
  targets         = discovery.relabel.alloy_pods.output
  forward_to      = [prometheus.remote_write.mimir.receiver]
  job_name        = "alloy"
  scrape_interval = "15s"
}
```

---

## 핵심 알림 규칙

```yaml
# PrometheusRule or Mimir alerting rule
groups:
  - name: alloy.rules
    rules:
      # Alloy Pod 다운
      - alert: AlloyDown
        expr: up{job="alloy"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Alloy 인스턴스 다운: {{ $labels.pod }}"
          description: "{{ $labels.cluster }}의 Alloy Pod {{ $labels.pod }}가 1분 이상 응답 없음"

      # Remote Write 실패율 증가
      - alert: AlloyRemoteWriteFailures
        expr: |
          rate(prometheus_remote_storage_failed_samples_total[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Alloy Remote Write 실패 발생"
          description: "{{ $labels.pod }}에서 지난 5분간 샘플 전송 실패 발생"

      # 전송 대기 샘플 증가 (백엔드 지연 징후)
      - alert: AlloyRemoteWritePending
        expr: |
          prometheus_remote_storage_samples_pending > 50000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Alloy Remote Write 큐 적체"
          description: "{{ $labels.pod }}의 전송 대기 샘플이 50,000 초과"

      # 스크랩 실패율 증가
      - alert: AlloyScrapeFailed
        expr: |
          rate(prometheus_scrape_duration_seconds_count{scrape_job!=""}[5m]) > 0
          and
          rate(up[5m]) == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Alloy 스크랩 실패"
          description: "{{ $labels.pod }}의 스크랩 타겟 {{ $labels.instance }} 접근 실패"

      # 로그 드롭 발생
      - alert: AlloyLogsDropped
        expr: |
          rate(loki_process_dropped_lines_total[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Alloy 로그 드롭 발생"
          description: "{{ $labels.pod }}에서 로그가 드롭되고 있음"

      # OTel Span 전송 실패
      - alert: AlloyTracesExportFailed
        expr: |
          rate(otelcol_exporter_send_failed_spans[5m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Alloy 트레이스 전송 실패"
          description: "{{ $labels.pod }}에서 span 전송 실패 발생"
```

---

## Grafana 대시보드

Grafana Labs에서 제공하는 공식 대시보드:

| 대시보드 | Grafana Dashboard ID | 설명 |
|---|---|---|
| Alloy Overview | 20021 | 전체 상태 요약 |
| Prometheus Components | 20022 | 메트릭 수집 파이프라인 상태 |
| Loki Components | 20023 | 로그 수집 파이프라인 상태 |
| OTel Components | 20024 | 트레이스 파이프라인 상태 |

Grafana에서 Import:
```
Dashboards → Import → ID 입력 → Load
```

---

## Web UI 활용

```bash
kubectl port-forward svc/alloy 12345:12345 -n monitoring
```

- `http://localhost:12345` → **Graph** 탭: 파이프라인 컴포넌트 연결 시각화
- `http://localhost:12345/component` → **Components** 탭: 각 컴포넌트 상태 및 exports 실시간 확인

컴포넌트 상태 예시:
```
prometheus.scrape.pods
  Status: RUNNING
  Health: healthy

  Arguments:
    targets: 42 targets
    scrape_interval: 30s

  Exports:
    (none)

  Debug info:
    Last scrape: 15s ago
    Scrape duration: 234ms
```

---

## 로그 기반 모니터링

```bash
# 실시간 Alloy 로그 모니터링
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy -f

# 에러 로그만 필터링
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep -E "level=error|level=warn"

# 특정 컴포넌트 로그
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy | grep "prometheus.scrape"
```

---

## 다음 단계

- [트러블슈팅 가이드](../04-operations/troubleshooting-guide.md)
- [End-to-End 실습](../05-practice/e2e-practice.md)
