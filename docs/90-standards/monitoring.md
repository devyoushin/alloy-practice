# 모니터링 규칙 — alloy-practice

## Grafana Alloy 자체 모니터링

### 핵심 메트릭

```promql
# 수집 타겟 수
count(alloy_component_controller_running_components{state="running"})

# 컴포넌트 오류 수
sum(alloy_component_controller_running_components{state="unhealthy"})

# 메트릭 전송 성공률
rate(prometheus_remote_write_sent_bytes_total[5m])

# 로그 전송 속도
rate(loki_write_sent_bytes_total[5m])

# 메모리 사용량
process_resident_memory_bytes{job="alloy"}

# Span 수신 속도
rate(otelcol_receiver_accepted_spans_total[5m])
```

### 알림 규칙 (권장)

| 알림 | 조건 | 심각도 |
|------|------|--------|
| AlloyDown | `up{job="alloy"} == 0` | critical |
| AlloyComponentUnhealthy | unhealthy 컴포넌트 존재 | warning |
| AlloyExportFailing | 전송 실패 지속 | warning |
| AlloyMemoryHigh | 메모리 > 제한 80% | warning |

## SLO 정의 (Alloy 서비스)

| SLI | 목표 | 측정 방법 |
|-----|------|---------|
| 컴포넌트 가용성 | 99.9% | unhealthy 컴포넌트 비율 |
| 메트릭 전송 성공률 | 99.5% | remote_write 실패율 |
| 로그 전송 성공률 | 99.5% | loki write 실패율 |

## 대시보드 구성 (Grafana)
- **Overview**: 컴포넌트 상태, 처리 속도
- **Metrics Pipeline**: 스크레이핑 현황, remote_write 상태
- **Logs Pipeline**: 로그 수집 속도, Loki 전송 상태
- **Traces Pipeline**: Span 수신/전송 현황
- **Resources**: CPU, 메모리, 네트워크 I/O
