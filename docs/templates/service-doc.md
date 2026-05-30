# [서비스명] — Grafana Alloy 파이프라인 문서

> 작성일: YYYY-MM-DD | 작성자: | 검토자:

## 개요
<!-- 이 Alloy 파이프라인의 목적과 수집 대상 -->

## 구성 정보

| 항목 | 값 |
|------|-----|
| Alloy 버전 | |
| 배포 모드 | DaemonSet / Deployment |
| 수집 신호 | Metrics / Logs / Traces / Profiles |
| 클러스터 모드 | 활성화 / 비활성화 |

## 컴포넌트 파이프라인

```
[수집 단계]                [처리 단계]              [전송 단계]
discovery.kubernetes  -->  prometheus.relabel  -->  prometheus.remote_write
loki.source.kubernetes --> loki.process       -->  loki.write
otelcol.receiver.otlp -->  otelcol.processor  -->  otelcol.exporter
```

## River 설정 요약

```river
// 핵심 컴포넌트 설정 요약
```

## 운영 체크리스트
- [ ] Alloy UI (`http://<alloy>:12345`) 컴포넌트 상태 확인
- [ ] 메트릭 전송 정상 확인
- [ ] 로그 수집 정상 확인
- [ ] 메모리 사용량 정상 범위 확인

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| | | |

## 관련 문서
- 런북:
- Alloy River 설정 파일:
- Grafana 대시보드:
