# Alloy 설정 언어 가이드

Grafana Alloy는 **Alloy 설정 언어(River 기반)** 를 사용합니다.
`.alloy` 확장자 파일에 컴포넌트를 선언하여 파이프라인을 구성합니다.

---

## 기본 구조

### 컴포넌트 선언

```alloy
// 형식: NAMESPACE.TYPE "LABEL" { ... }
prometheus.scrape "my_scraper" {
  targets    = [{ "__address__" = "localhost:9090" }]
  forward_to = [prometheus.remote_write.default.receiver]
}
```

- **NAMESPACE**: 기능 그룹 (`prometheus`, `loki`, `discovery`, `otelcol` 등)
- **TYPE**: 컴포넌트 종류 (`scrape`, `remote_write`, `kubernetes` 등)
- **LABEL**: 같은 TYPE 내에서 인스턴스를 구분하는 이름 (고유해야 함)

컴포넌트의 전체 참조 이름: `NAMESPACE.TYPE.LABEL` (예: `prometheus.scrape.my_scraper`)

---

## 속성 (Attributes)

```alloy
prometheus.remote_write "mimir" {
  // 문자열
  url = "http://mimir:8080/api/v1/push"

  // 불리언
  send_native_histograms = true

  // 숫자
  max_retries = 3

  // Duration
  scrape_interval = "30s"

  // 리스트
  targets = [
    { "__address__" = "app1:8080" },
    { "__address__" = "app2:8080" },
  ]
}
```

---

## 참조 (References)

컴포넌트의 exports를 다른 컴포넌트의 입력으로 연결합니다.

```alloy
// discovery.kubernetes가 발견한 타겟을 prometheus.scrape에 연결
prometheus.scrape "pods" {
  targets    = discovery.kubernetes.pods.targets   // 참조
  forward_to = [prometheus.remote_write.mimir.receiver]  // 참조
}
```

참조 형식: `NAMESPACE.TYPE.LABEL.EXPORT_NAME`

자주 쓰는 exports:
| 컴포넌트 | Export | 설명 |
|---|---|---|
| `discovery.kubernetes` | `.targets` | 발견된 스크랩 타겟 목록 |
| `discovery.relabel` | `.output` | 릴레이블링된 타겟 |
| `prometheus.remote_write` | `.receiver` | 메트릭 수신기 |
| `loki.write` | `.receiver` | 로그 수신기 |
| `otelcol.exporter.otlp` | `.input` | OTel 데이터 수신기 |

---

## 블록 (Blocks)

블록은 컴포넌트 안에 중첩된 설정 그룹입니다.

```alloy
prometheus.remote_write "mimir" {
  // 최상위 속성
  wal_dir = "/var/lib/alloy/wal"

  // endpoint 블록 (중첩 설정)
  endpoint {
    url = "http://mimir:8080/api/v1/push"

    // 블록 안에 블록
    basic_auth {
      username = "admin"
      password = "secret"
    }

    // 또 다른 중첩 블록
    queue_config {
      capacity = 10000
      max_shards = 50
    }
  }
}
```

---

## 표현식 (Expressions)

### 환경 변수 참조

```alloy
prometheus.remote_write "mimir" {
  endpoint {
    url = env("MIMIR_URL")   // 환경 변수 읽기
  }
}
```

### 문자열 보간

```alloy
discovery.kubernetes "pods" {
  role = "pod"

  namespaces {
    own_namespace = false
    names = [env("MY_NAMESPACE")]
  }
}
```

### 조건부 표현식

```alloy
// 삼항 연산자 지원
local.file "config" {
  filename = env("CONFIG_PATH") != "" ? env("CONFIG_PATH") : "/etc/alloy/config.alloy"
}
```

---

## 자주 쓰는 내장 함수

```alloy
// 환경 변수 읽기
env("VARIABLE_NAME")

// JSON 파싱
json_decode(string)

// Base64 디코딩
base64_decode(string)

// 문자열 변환
string(value)

// 파일 읽기
file.path("/etc/alloy/secret.txt")
```

---

## Secret 처리

```alloy
// 방법 1: 환경 변수로 주입
prometheus.remote_write "mimir" {
  endpoint {
    url = env("MIMIR_URL")
    basic_auth {
      username = env("MIMIR_USER")
      password = env("MIMIR_PASS")
    }
  }
}

// 방법 2: Kubernetes Secret을 ConfigMap에서 로드
remote.kubernetes.secret "creds" {
  name      = "alloy-secrets"
  namespace = "monitoring"
}

prometheus.remote_write "mimir" {
  endpoint {
    url = remote.kubernetes.secret.creds.data["mimir_url"]
  }
}
```

---

## 여러 파일 분리 (모듈화)

대규모 설정은 파일을 나눠 관리합니다.

```bash
# 파일 구조 예시
config/
├── main.alloy          # 메인 (import 선언)
├── metrics.alloy       # 메트릭 파이프라인
├── logs.alloy          # 로그 파이프라인
└── traces.alloy        # 트레이스 파이프라인
```

```alloy
// main.alloy
import.file "metrics" {
  filename = "/etc/alloy/modules/metrics.alloy"
}

import.file "logs" {
  filename = "/etc/alloy/modules/logs.alloy"
}
```

---

## 설정 디버깅

### 문법 검사

```bash
# alloy binary로 문법 검사
alloy fmt config.alloy         # 포맷 정리
alloy run config.alloy --dry-run  # 실행 없이 검증
```

### Web UI에서 확인

```bash
kubectl port-forward svc/alloy 12345:12345 -n monitoring
# http://localhost:12345 접속 후:
# - Graph: 파이프라인 시각화
# - Components: 각 컴포넌트의 현재 상태 및 exports 값
```

### 로그 레벨 조정

```alloy
logging {
  level  = "debug"   // debug, info, warn, error
  format = "logfmt"  // logfmt, json
}
```

---

## 설정 예시: 전체 구조 패턴

```alloy
// 1. 로깅 설정 (선택)
logging {
  level = "info"
}

// 2. 디스커버리 (Source)
discovery.kubernetes "pods" {
  role = "pod"
}

// 3. 릴레이블링 (Transform)
discovery.relabel "pods" {
  targets = discovery.kubernetes.pods.targets

  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
}

// 4. 수집 (Collect)
prometheus.scrape "pods" {
  targets    = discovery.relabel.pods.output
  forward_to = [prometheus.remote_write.mimir.receiver]
}

// 5. 전송 (Export)
prometheus.remote_write "mimir" {
  endpoint {
    url = "http://mimir-nginx.monitoring.svc:8080/api/v1/push"
  }
}
```

---

## 다음 단계

- [Kubernetes 서비스 디스커버리](./kubernetes-discovery-guide.md)
- [메트릭 수집 파이프라인](./metrics-guide.md)
