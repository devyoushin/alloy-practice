# Alloy systemd 설치

단일 VM이나 베어메탈에서 Grafana Alloy를 systemd 서비스로 관리할 때 사용한다.

## 대상

- `alloy.service`

## 준비물

- Alloy 바이너리
- 설정 파일: `/etc/alloy/config.alloy`
- 데이터/상태 디렉터리: `/var/lib/alloy`

## 절차

1. 바이너리를 설치한다.
2. 설정 파일을 배치한다.
3. systemd unit 파일을 `/etc/systemd/system/alloy.service`에 둔다.
4. `systemctl enable --now alloy`로 시작한다.
5. `journalctl -u alloy -f`로 로그를 본다.

## 확인 명령

```bash
systemctl status alloy
curl http://localhost:12345/ready
curl http://localhost:12345/metrics | grep alloy_build_info
```

## 운영 포인트

- River 설정 문법이 맞는지 먼저 검증한다.
- metrics/logs/traces export endpoint를 분리해서 관리한다.
- reload 가능 여부를 확인한 뒤 `/-/reload` 흐름을 사용한다.
