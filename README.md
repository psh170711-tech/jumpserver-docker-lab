# jumpserver-docker-lab

macOS에서 Docker로 JumpServer를 배포하고 구동 상태와 포트 매핑을 확인한 실습

맥 Docker Desktop 위에 오픈소스 PAM(Privileged Access Management) 솔루션 **JumpServer**를 구축하고,
같은 네트워크의 대상 서버로 접속을 통제·감사하는 실습 환경입니다.

## 구성

```
 [브라우저] ──127.0.0.1:8080──▶ [pam-jumpserver]  ──SSH──▶ [mcx-test]
                                 jms_all v4.10.20           ubuntu 24.04
                                 └─ 계정·권한 관리 / 명령어 통제 / 세션 녹화·감사
                         ─────────── docker network: pam-lab-net ───────────
```

| 컨테이너 | 이미지 | 역할 |
|---|---|---|
| `pam-jumpserver` | `jumpserver/jms_all:v4.10.20` | 점프 서버 (웹 콘솔, 내장 DB·Redis 포함 올인원) |
| `mcx-test` | `ubuntu:24.04` | 점프 서버를 거쳐 접속하는 대상 서버 |

- 웹 콘솔은 `127.0.0.1`에만 바인딩해 외부에서 직접 접근할 수 없게 했습니다.
- 데이터는 `pam-jsdata`, `pam-pgdata` 볼륨에 보존되어 컨테이너를 지워도 유지됩니다.

## 실행

```bash
cp .env.example .env
# .env 의 SECRET_KEY, BOOTSTRAP_TOKEN 채우기 (openssl rand -hex 32)
docker compose up -d
```

브라우저에서 http://localhost:8080 접속.

## 구축 검증 (2026-11-24)

맥 Docker Desktop에서 실제로 기동하여 확인한 결과입니다. 전체 기록: [`docs/verify.txt`](docs/verify.txt)

| 항목 | 결과 |
|---|---|
| Docker | Client 28.5.2 / Server 28.5.2 |
| 컨테이너 | `pam-jumpserver` — `jumpserver/jms_all:v4.10.20`, **Up** |
| 포트 매핑 | `127.0.0.1:8080 -> 80/tcp` (로컬 전용 바인딩) |
| 웹 콘솔 | `curl -sI http://127.0.0.1:8080` → **HTTP/1.1 200 OK** |

컨테이너 내부에서 확인한 구성 요소:

| 프로세스 | 역할 |
|---|---|
| `nginx` | 웹 콘솔 리버스 프록시 |
| `gunicorn`, `celery` | Core API(Django) 및 비동기 작업 |
| `postgres`, `redis-server` | 내장 DB / 캐시·세션 |
| `koko` | SSH·웹 터미널 접속 게이트웨이 |
| `lion`, `guacd` | RDP·VNC 원격 데스크톱 게이트웨이 |
| `supervisord` | 컨테이너 내부 프로세스 관리 |

## 실습 내용

- [ ] 사용자·자산(mcx-test)·계정 등록 및 권한 부여
- [ ] 명령어 필터(위험 명령 차단) 설정
- [ ] 세션 녹화·재생, 로그인/명령 감사 로그 확인
- [ ] (이후) HA 구성

## 보안 참고

`.env`와 `docker inspect` 원본은 비밀값을 담고 있어 `.gitignore`로 제외합니다.
