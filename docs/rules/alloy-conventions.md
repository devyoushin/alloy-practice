# Grafana Alloy 컨벤션 규칙

## River 설정 파일 구조
```
config/
├── metrics.alloy      # 메트릭 파이프라인
├── logs.alloy         # 로그 파이프라인
├── traces.alloy       # 트레이스 파이프라인
└── common.alloy       # 공통 컴포넌트 (autodiscovery 등)
```

## 컴포넌트 네이밍 규칙
- 형식: `{type}.{purpose}` 라벨 사용
- 예: `prometheus.scrape "kubernetes_pods"`, `loki.write "default"`
- 레이블은 snake_case, 목적을 명확히 표현

## 파이프라인 작성 규칙
```river
// 1. Discovery (자동 검색)
discovery.kubernetes "pods" {
  role = "pod"
}

// 2. Transform (레이블 변환)
prometheus.relabel "drop_high_cardinality" {
  forward_to = [prometheus.remote_write.default.receiver]
  // 규칙 작성
}

// 3. Export (전송)
prometheus.remote_write "default" {
  endpoint {
    url = env("PROMETHEUS_URL")
  }
}
```

## 자격증명 관리
```river
// 올바른 예 — 환경변수 참조
prometheus.remote_write "default" {
  endpoint {
    url = env("REMOTE_WRITE_URL")
    basic_auth {
      username = env("REMOTE_WRITE_USER")
      password = env("REMOTE_WRITE_PASSWORD")
    }
  }
}

// 금지 — 하드코딩
// password = "my-secret"  // 절대 금지
```

## 레이블 정규화 규칙
- Kubernetes 자동 검색 레이블: `__meta_kubernetes_*` → 표준 레이블로 변환
- 불필요 레이블 `drop` action으로 제거
- `job` 레이블: `{namespace}/{service}` 형식

## 배포 모드 선택
| 모드 | 사용 사례 |
|------|---------|
| DaemonSet | 노드별 로그/메트릭 수집 |
| Deployment | 중앙 집중 처리 |
| Sidecar | 특정 파드 전용 수집 |
