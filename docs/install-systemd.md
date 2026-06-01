# Alloy systemd 설치

단일 VM이나 베어메탈에서 Grafana Alloy를 systemd 서비스로 관리할 때 사용한다.

## 대상

- `alloy.service`

## 준비물

- Alloy 바이너리
- 설정 파일: `/etc/alloy/config.alloy`
- 데이터/상태 디렉터리: `/var/lib/alloy`

## RPM 설치

RHEL 계열에서는 Alloy RPM 또는 Grafana repository를 통해 설치할 수 있다. 운영 환경에서는 repository와 버전을 고정한다.

### RPM 파일 직접 설치

```bash
ALLOY_VERSION="<VERSION>"
sudo dnf install -y "./alloy-${ALLOY_VERSION}-1.x86_64.rpm"
```

### repository 등록 후 설치

```bash
sudo tee /etc/yum.repos.d/grafana.repo >/dev/null <<'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

sudo dnf install -y alloy
rpm -qa | grep alloy
systemctl cat alloy
```

## tarball 설치

패키지 저장소를 쓰지 않는 환경에서는 릴리스 tarball이나 바이너리를 `/usr/local/bin/alloy`에 배치한다.

```bash
ALLOY_VERSION="<VERSION>"
curl -LO "https://github.com/grafana/alloy/releases/download/v${ALLOY_VERSION}/alloy-linux-amd64.zip"
unzip alloy-linux-amd64.zip
sudo install -m 0755 alloy-linux-amd64 /usr/local/bin/alloy
sudo mkdir -p /etc/alloy /var/lib/alloy
```

## 절차

1. RPM 또는 tarball 방식 중 하나로 바이너리를 설치한다.
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
