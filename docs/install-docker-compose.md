# Alloy Docker Compose 설치

로컬에서 Alloy 파이프라인을 빠르게 검증할 때 사용한다.

## 대상

- `alloy`

## 절차

1. `compose.yaml`에 이미지와 포트를 정의한다.
2. River 설정 파일을 mount 한다.
3. `docker compose up -d`로 올린다.
4. `docker compose logs -f`와 `docker compose ps`를 확인한다.

## 확인 명령

```bash
docker compose ps
curl http://localhost:12345/ready
```

## 운영 포인트

- 로컬 실험은 `alloy`와 backend endpoint만 확인해도 충분하다.
- 운영 환경은 Helm 문서를 우선한다.
