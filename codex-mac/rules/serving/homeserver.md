# homeserver 서빙 환경

2026-10-27 KST에 Tailscale SSH로 읽기 전용 확인. 실행 상태는 당시 관찰이며 작업 전에 다시 확인한다.

## 적용 범위와 접속
- 이 Mac은 개발·문서 작업 환경이다. 서버의 경로를 Mac의 기본 작업 경로로 적용하지 않는다.
- 서빙 작업 전에 이 문서와 같은 폴더의 operations.md, practices.md를 읽는다.
- 과거 샌드박스 원문은 legacy-sandbox.md에 보관한다. 과거 경로·권한·7일 백업 정책·adminctl 정책을 현재 정책으로 적용하지 않는다.
- Tailscale 노드: `homeserver`, IPv4: `100.64.150.42`. 접속: `ssh user@homeserver`.
- OS 호스트명은 `server`다. Tailscale 노드 이름과 구분한다.
- 로그인 계정은 `user`(UID/GID 1000), 홈은 `/home/user`, sudo·docker 그룹 소속이다.
- `sudo -n true`는 비밀번호 필요로 실패했다. 비대화형 sudo 가능 또는 root 권한을 가정하지 않는다. 비밀번호를 수집·저장하지 않는다.
- `deploy`(UID/GID 1001)의 홈은 `/home/deploy`, 로그인 셸은 `/srv/infra/scripts/ssh-entry.sh`다. 일반 조사용 셸로 사용하지 않는다.
- 확인 당시 `/home/user/AGENTS.md`, `/home/user/.codex/AGENTS.md`, `/srv/infra/AGENTS.md`, `/srv/infra/CLAUDE.md`는 발견되지 않았다. 작업할 프로젝트의 지침은 별도로 확인한다.

## Tailscale 접근 조건
- Mac의 Tailscale IP는 `100.69.94.101`, 태그는 `tag:mac`; 서버는 `tag:homeserver`다.
- 태그 장치는 사용자 ID 대신 태그에 대한 접근 규칙을 사용한다.
- Mac에서 서버 접속에는 grants의 `tag:mac` → `tag:homeserver`, `tcp:22`와 ssh의 같은 출발·대상에 대한 `users: ["user"]` 허용이 필요하다.
- 사용자가 두 규칙을 저장한 후 장치 목록 표시와 `ssh user@homeserver` 성공을 확인했다. 정책 전체를 API로 조회한 것은 아니다.
- CI는 사용자 제공 정책과 인프라 문서상 `tag:ci`에서 `deploy`로 접속한다. 실제 CI 실행은 이번에 검증하지 않았다.
- `tailscale status`는 이 클라이언트에 보이는 장치 목록이다. 목록 누락만으로 서버 부재나 다른 tailnet으로 단정하지 않는다.

## 현재 경로
| 경로 | 용도와 확인 범위 |
|---|---|
| `/srv/infra` | 현재 인프라 저장소·배포 정의·스크립트. 소유자는 deploy |
| `/srv/infra/platform` | 공통 Compose 정의: `database.yml`, `edge.yml` |
| `/srv/infra/apps/<서비스>` | 앱 Swarm 정의 `stack.yml`, 서비스 메타데이터 `service.yaml` |
| `/srv/infra/configs/nginx/conf.d` | nginx에 직접 마운트되는 설정 |
| `/srv/secrets` | 비밀 설정. deploy 소유, 디렉터리 0700. 값과 내부 파일은 이번에 읽지 않음 |
| `/srv/data` | 현재 데이터. PostgreSQL과 admin·ggultarae·memo 경로 확인 |
| `/srv/state` | releases, locks, admin 상태, restore-tests.log |
| `/srv/backups` | 임시 백업, deploy 소유 0700. 내부 백업 파일은 미열람 |
| `/srv/logs` | 로그 루트. nginx 로그 디렉터리 확인 |
| `/srv/recovery/legacy-20261026` | root 전용 복구 자료. 내용 미확인, 임의 정리 금지 |
| `/mnt/vault` | `/dev/sda1` ext4 마운트. data·backup 디렉터리 존재 |

- `/srv/data`와 `/srv/backups`는 `/dev/nvme0n1p2`의 루트 파일시스템에 있다. `/mnt/vault/data`를 현재 운영 데이터로 간주하지 않는다.
- `/srv/infra/infra`에도 이전 구조의 파일들이 존재한다. 현재 실행 정의는 바깥 `/srv/infra/platform`, `/srv/infra/apps`, `/srv/infra/scripts`에서 확인했다. 안쪽 경로의 현재 사용 여부는 미확인이다.
- `/srv/infra`의 Git 조회는 소유자 불일치(dubious ownership)로 거부됐다. safe.directory를 추가하거나 소유권을 바꾸지 않았다. 현재 브랜치·커밋·미커밋 변경은 미확인이다.
- 신규 개발·worktree 경로는 서버에 일괄 지정하지 않는다. 앱 개발과 빌드는 기존 프로젝트·CI 경로를 따른다.

## 실행 구성
- 공통 플랫폼은 Compose: `infra-db`(postgres), `infra-edge`(nginx·cloudflared).
- 앱은 Swarm 스택: `app-admin`, `app-blog`, `app-ggultarae`, `app-memo`, `app-secret`.
- 조회 당시 앱 서비스 8개는 모두 replicas 1/1이었다. 실제 사용자 기능을 검증한 결과는 아니다.
- nginx·PostgreSQL 및 healthcheck가 정의된 앱 컨테이너들은 healthy로 표시됐다. cloudflared 등 healthcheck가 없는 컨테이너는 실행 상태만 확인했다.
- 이미지: PostgreSQL 17.11-alpine, nginx 1.31.8-alpine, cloudflared 2026.10.3. 앱 서비스는 GHCR digest로 고정되어 있었다.
- 네트워크: `edge`, `db_admin`, `db_ggultarae`, `db_memo`, `db_secret`은 Swarm overlay. `postgres_host`는 bridge다.
- PostgreSQL bind: `/srv/data/postgres` → `/var/lib/postgresql/data`; 비밀번호 파일 경로는 `/srv/secrets/postgres_password`.
- DB는 Docker 네트워크와 호스트 `127.0.0.1:5432`에 연결된다. 외부 공개 포트라고 해석하지 않는다.
- nginx 설정과 로그는 각각 `/srv/infra/configs/nginx/conf.d`, `/srv/logs/nginx`에서 bind mount한다.
- 인프라 문서상 외부 요청은 Cloudflare Tunnel → nginx → 앱으로 전달된다. Cloudflare 대시보드와 방화벽 전체 설정은 미확인이다.

## 서비스와 검증 정의
다음 URL은 `apps/*/service.yaml`의 정의다. 이번에 HTTP·로그인·쓰기 기능을 실행하지 않았다.

| 스택 | 실행 서비스 | 공개 URL |
|---|---|---|
| app-admin | web | `https://admin.sjh354.xyz/api/health` |
| app-blog | blog | `https://sjh354.xyz/` |
| app-ggultarae | web, api, worker | `https://ggultarae.cc/`, `https://api.ggultarae.cc/v1/health` |
| app-memo | memo, collab | `https://memo.sjh354.xyz/` |
| app-secret | secret | `https://secret.sjh354.xyz/` |

- 내부 점검 주소와 DB·역할·마이그레이션 설정의 원본은 각 `service.yaml`이다.
- `admin-collector`, `admin-runner`는 active, `admin-agent`는 inactive였다. 선택적 에이전트의 비활성을 장애로 단정하지 않는다.

## 배포와 잠금
- 서버 README상 앱 빌드·테스트·이미지 게시는 앱 저장소의 GitHub Actions가 담당한다. 서버는 실행·데이터 보관을 담당한다.
- 인프라 워크플로: `.github/workflows/deploy-service.yml`, `rollback-service.yml`, `infra-release.yml`, `validate.yml`.
- 배포 진입점: `/srv/infra/scripts/deploy.sh <서비스> <40자리 SHA>`, 롤백: `scripts/rollback.sh <서비스> [SHA]`.
- CI용 SSH 셸은 `deploy <서비스> <SHA>`, `rollback <서비스> [SHA]`, `infra [edge]`만 허용하도록 구현되어 있다. 조사 명령을 이 경로로 우회하지 않는다.
- 플랫폼 변경 진입점은 `/srv/infra/apply-platform.sh`. DB 재생성은 별도 운영 영향과 점검 시간을 확인한다.
- 실제 서비스 변경 전 해당 저장소 지침과 현재 스크립트를 읽는다. 이번 문서 작업에서 배포·롤백·DB 변경은 실행하지 않았다.
- 서비스 잠금: `/srv/state/locks/<서비스>.lock`, 전역 잠금: `/srv/state/locks/global.lock`; 스크립트의 flock 대기 한도는 300초다.
- `preflight.sh`는 데이터 디스크 사용률 80%부터 경고, 90% 이상에서 배포를 중단하도록 구현되어 있다.
- 검증 진입점: `scripts/healthcheck.sh [서비스]`. 배포 시 서비스 수렴·이미지·내부 HTTP·공개 URL을 확인하고 관련 실제 기능을 추가 확인한다.
- memo는 service.yaml에 배포 중 마이그레이션이 정의되어 있다. ggultarae·secret 등 manual 설정은 이미지 배포만으로 스키마 적용을 가정하지 않는다.
- 이미지 롤백은 DB 스키마를 되돌리지 않는다. admin은 별도 호스트 설치 단계도 있으므로 해당 스크립트를 확인한다.

## 백업·복원
- `infra-backup.timer`: enabled, 매일 04:00 Asia/Seoul, root로 `/srv/infra/scripts/backup.sh` 실행.
- 최근 백업 서비스 상태는 Result=success, ExecMainStatus=0, 종료 시각 2026-10-27 08:55:33 KST였다.
- 스크립트는 DB별 pg_dump와 globals를 `/srv/backups/<UTC시각>`에 생성하고, `/srv/data`(postgres 원시 디렉터리 제외)·`/srv/secrets`와 함께 restic으로 전송한다.
- 외부 전송 스냅샷 확인 후 `--keep-daily 30` 정책과 2% 데이터 검사를 실행하도록 구현되어 있다. 로컬은 체크섬 파일이 있는 3일 초과 백업 디렉터리를 전송 성공 후 정리한다.
- restic 설정 원본 경로는 `/srv/secrets/restic.env`다. 실제 외부 저장소·스냅샷·내용은 직접 검증하지 않았다. 서비스 종료 성공과 복구 가능 판정을 구분한다.
- `infra-restore-test.timer`: enabled, 매월 1일 05:00 Asia/Seoul, root로 `/srv/infra/scripts/restore.sh test` 실행.
- 복원 검사 기록 경로는 `/srv/state/restore-tests.log`. 이번에 내용을 열람하거나 복원 검사를 실행하지 않았다.
- 복구 절차 원본: `/srv/infra/docs/disaster-recovery.md`, `scripts/restore.sh`. 복원은 데이터 영향과 승인 범위를 확인한 뒤 수행한다.
- 루트 디스크의 임시 복사본만으로 외부 백업을 대체하지 않는다. 과거 7일 보존 정책을 현재 백업에 적용하지 않는다.

## 문서 기준과 남은 확인
- 현재 서버 설명의 기준: `/srv/infra/README.md`, `docs/deployment.md`, `docs/operations.md`, `docs/disaster-recovery.md`, 실제 scripts·platform·service.yaml·systemd 상태.
- 서버 문서에도 오래된 설명이 있다. 예: docs/operations.md의 DB 내부 전용 설명과 현재 localhost 바인딩, docs/deployment.md의 admin 미병합 안내와 현재 실행 상태. 현재 브랜치·병합 여부까지 확인한 것은 아니다.
- admin 서비스 존재와 collector·runner 실행은 확인했다. 예전 adminctl 위치·토큰·중앙 권한 요청 절차는 확인하지 않았으므로 재사용하지 않는다.
- 운영 변경 로그의 정확한 위치·정책, admin 검토 흐름, 외부 백업·복원 검사 이력, 현재 Git 기준점, 실제 CI 배포 결과는 추가 확인이 필요하다.
- 이번 변경은 Mac의 규칙 문서에만 적용했다. 원격 서버 파일·서비스·권한·네트워크 설정은 변경하지 않았다.

## 중앙 로그 (2026-10-28 추가)

- 구성 원본: `/srv/infra/platform/logging.yml`, `configs/vector/vector.yaml`, `configs/loki/loki.yaml`.
- Compose 프로젝트 `infra-logging`: Vector → Loki. 전용 bridge, Loki API는 서버 `127.0.0.1:3100`에만 게시한다.
- 데이터: `/srv/data/loki`(10001:10001, 0750), `/srv/data/vector`(root:root, 0700).
- Loki 14일 보존, Vector 디스크 버퍼 1GiB. 두 경로는 restic 앱 데이터 백업에서 제외한다. 로그 재해 복구·완전 무손실은 보장하지 않는다.
- 수집 대상: 컨테이너 stdout/stderr와 nginx `access_host.log`·`error.log`. admin systemd·앱 별도 파일 로그는 자동 수집 대상이 아니다.
- Mac MCP 이름 `loki-local`, 패키지 `simple-loki-mcp@0.0.9`(HTTP API 직접 사용). 사용자 설정은 `~/.codex/config.toml`이다.
- Mac/서버 logcli: 각각 `~/.local/bin/logcli`, 공식 3.7.9 릴리스·SHA256 검증.
- 조회 전 필요 시 터널: `ssh -N -L 127.0.0.1:3100:127.0.0.1:3100 -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3 user@homeserver`.
- 라벨: `host="homeserver"`, `project`, `service`, `stream`. Swarm 서비스 예: `service="app-memo_memo"`.
- 운영 절차: `/srv/infra/docs/logging.md`. 설치·검증 기록: `/srv/state/logging/`.
- Docker 소켓의 :ro 마운트는 API 읽기 전용 권한이 아니다. Vector는 호스트 관리자에 준하는 신뢰가 필요하다.

- 2026-10-28 배포 확인: main `0e05b2d`, Infra release `40679866526` 성공. nginx 파일 수집·Loki 중단 후 재전송·logcli/MCP 실제 조회 통과.
- Mac에서 이번에 실행한 터널의 제어 소켓은 `~/.ssh/loki-homeserver.control`이다. `ssh -S ~/.ssh/loki-homeserver.control -O check user@homeserver`로 재사용 여부를 확인한다. 자동 시작은 설정하지 않았다.
- 네트워크는 Docker 29에서 internal bridge 포트 게시가 생성되지 않아 전용 일반 bridge로 확정했다. Loki는 127.0.0.1에만 게시한다.
