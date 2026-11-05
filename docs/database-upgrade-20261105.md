# 원본 병합 후 DB 업그레이드

이 문서는 저장소의 `compose.yml`에 있는 PostgreSQL 17, 서비스 `wakapi`/`db`, DB `wakapi`를 기준으로 한다. 서버에서 다른 Compose 파일·프로젝트 이름을 사용한다면 아래 명령에도 기존 `-f`, `-p`, `--env-file` 옵션을 똑같이 사용한다. 다른 프로젝트로 실행하면 기존 DB 볼륨 대신 새 볼륨을 만들 수 있다.

## 동작 방식

별도의 SQL 파일을 순서대로 실행할 필요는 없다. 새 Wakapi가 시작할 때 `main.go`에서 `migrations.Run`을 호출하고, 사전 데이터 정리 → GORM 스키마 갱신 → 후속 데이터 보정을 실행한다. `WAKAPI_SKIP_MIGRATIONS`는 `false`, `WAKAPI_DB_AUTOMIGRATE_FAIL_SILENTLY`는 `false`로 둔다. 앱 DB 계정에는 테이블·인덱스·제약조건을 변경할 권한이 필요하다.

`scripts/dbmigrate.go`는 SQLite/PostgreSQL/MySQL 사이의 데이터 이전 도구다. 이번처럼 기존 PostgreSQL을 계속 사용하는 버전 업그레이드에는 실행하지 않는다. PostgreSQL 서버 버전과 기존 볼륨은 그대로 유지한다.

이번 변경에는 WebAuthn 테이블/사용자 필드, AI heartbeat 필드, 이메일·OIDC 값 정규화, 고아 duration 삭제, 인덱스 교체가 포함된다. 일부 작업은 백그라운드에서 진행되므로 `/api/health` 성공만으로 모든 데이터 보정이 완료됐다고 판단하지 않는다. 데이터 크기에 따라 인덱스 생성과 정리에 시간이 걸릴 수 있다.

## 1. 이전 이미지 보관과 새 이미지 빌드

운영 서버의 기존 Compose 프로젝트 디렉터리에서 실행한다. 셸 변수는 이후 단계에서도 사용하므로 같은 셸을 유지한다. 각 명령이 성공한 것을 확인한 뒤 다음 명령으로 진행한다.

```sh
backup_dir="$HOME/wakapi-backups/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"
chmod 700 "$backup_dir"
old_image=$(docker inspect --format '{{.Image}}' "$(docker compose ps -q wakapi)")
rollback_image="wakapi-rollback:pre-upstream-$(date +%Y%m%d-%H%M%S)"
docker image tag "$old_image" "$rollback_image"
printf '%s\n' "$rollback_image" > "$backup_dir/rollback-image.txt"
docker compose build wakapi
```

GHCR 이미지를 받아 실행하는 구성이라면 마지막 build 대신 새 커밋의 이미지를 받아 기존 image 설정에 반영한다. 이하 명령은 이 저장소의 `build: .` 구성을 기준으로 한다.

이전 이미지 태그는 같은 Docker 호스트에서의 롤백용이다. 환경 설정·비밀번호 salt·쿠키 키도 기존 값을 보관하고 그대로 사용한다.

## 2. 복제 DB에서 먼저 확인

운영 앱이 실행 중이어도 `pg_dump`는 일관된 스냅샷을 만든다. 복제 DB 이름이 이미 사용 중이면 다른 이름을 선택하고 아래 명령 전체에 반영한다. 기존 DB를 덮어쓰거나 삭제하지 않는다.

```sh
docker compose exec -T db sh -c 'exec pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' > "$backup_dir/rehearsal.dump"
docker compose exec -T db pg_restore --list < "$backup_dir/rehearsal.dump" > "$backup_dir/rehearsal.contents.txt"
docker compose exec -T db sh -c 'exec createdb -U "$POSTGRES_USER" -T template0 wakapi_migration_check'
docker compose exec -T db sh -c 'exec pg_restore -U "$POSTGRES_USER" -d wakapi_migration_check --no-owner --no-privileges --exit-on-error' < "$backup_dir/rehearsal.dump"
```

동일한 덤프를 복원할 수 있는지 이 단계에서 함께 확인한다. 새 앱을 실행하기 전에 복제 DB에서 다음 SQL로 기존 사용자·heartbeat 수를 기록하고, 중복 데이터를 확인한다. `psql` 접속 예시는 다음과 같다.

```sh
docker compose exec db sh -c 'exec psql -U "$POSTGRES_USER" -d wakapi_migration_check'
```

```sql
SELECT count(*) AS users FROM users;
SELECT count(*) AS heartbeats FROM heartbeats;
SELECT count(*) AS orphaned_durations
FROM durations d LEFT JOIN users u ON d.user_id = u.id WHERE u.id IS NULL;

SELECT email, count(*) FROM users
WHERE email IS NOT NULL AND email <> ''
GROUP BY email HAVING count(*) > 1;

SELECT lower(auth_type) AS provider, count(DISTINCT auth_type) AS variants
FROM users WHERE auth_type IS NOT NULL AND auth_type <> ''
GROUP BY lower(auth_type) HAVING count(DISTINCT auth_type) > 1;

SELECT lower(auth_type) AS provider, sub, count(*) FROM users
WHERE sub IS NOT NULL AND sub <> ''
GROUP BY lower(auth_type), sub HAVING count(*) > 1;
```

중복 확인 쿼리에서 행이 나오면 사용자 계정 관계를 확인한 후 정리한다. 특히 대소문자만 다른 OIDC 제공자 이름이 혼재하면 원본 마이그레이션은 오류로 중단한다. 사용자나 OIDC 연결을 임의로 삭제하지 않는다. `\q`로 psql을 종료한다.

복제 DB에 새 앱 한 개를 연결한다. 이 명령은 서비스 포트를 외부로 열지 않는다. 메일과 보관기간에 따른 데이터 삭제는 복제 검증 중 비활성화한다.

```sh
docker compose run -d --no-deps --name wakapi-migration-check \
  -e WAKAPI_DB_NAME=wakapi_migration_check \
  -e WAKAPI_SKIP_MIGRATIONS=false \
  -e WAKAPI_DB_AUTOMIGRATE_FAIL_SILENTLY=false \
  -e WAKAPI_MAIL_ENABLED=false \
  -e WAKAPI_DATA_RETENTION_MONTHS=-1 \
  -e WAKAPI_MAX_INACTIVE_MONTHS=-1 wakapi
docker logs -f wakapi-migration-check
```

`Ctrl+C`는 로그 보기만 종료한다. 로그에 `migration failed`가 없어야 한다. 서버가 뜬 다음 다음 명령과 아래 SQL 확인을 진행한다.

```sh
docker exec wakapi-migration-check /app/healthcheck
```

복제 DB에 다시 psql로 접속해 확인한다.

```sql
SELECT count(*) AS users FROM users;
SELECT count(*) AS heartbeats FROM heartbeats;
SELECT count(*) AS orphaned_durations
FROM durations d LEFT JOIN users u ON d.user_id = u.id WHERE u.id IS NULL;

SELECT key, value FROM key_string_values
WHERE key IN (
  '20260103-add_webauthn_fields',
  '20260214-nullify-empty-email',
  '20260214-nullify-empty-sub',
  '20260820-fix_ai_model_as_editor',
  '20260831-lowercase_oidc_provider_names',
  '20260925-drop_orphaned_durations',
  '20260929-drop_durations_time_index',
  '20261002-fix_neovim_wakatime_editor'
) ORDER BY key;

SELECT indexname, indexdef FROM pg_indexes
WHERE tablename = 'heartbeats' AND indexname = 'idx_user_project_time';
```

기존 포크 DB를 업그레이드한 경우 위 완료 표식 8개가 `done`, 새 인덱스가 존재하는지 확인한다. 표식이 아직 없으면 로그를 확인하고 백그라운드 작업이 끝날 때까지 기다린다. 인덱스 교체 마이그레이션 자체는 완료 표식을 기록하지 않으므로 인덱스를 직접 확인한다. 사용자·heartbeat 수는 유지되고 고아 duration은 0이어야 한다. 요약·duration은 재집계될 수 있어 개수가 같아야 하는 항목이 아니다.

검증이 끝나면 `docker stop wakapi-migration-check`로 복제 앱을 멈춘다. 복제 앱이 운영 DB를 사용하지 않았는지도 확인한다. 복제 DB와 컨테이너는 결과를 보관할 동안 남겨두어도 된다.

## 3. 운영 적용

복제 검증을 통과한 뒤 점검 시간에 진행한다. 여러 Wakapi 인스턴스가 같은 DB를 사용한다면 기존 인스턴스를 모두 중지하고, 최초 마이그레이션은 새 인스턴스 한 개로 실행한다. 프록시가 있다면 완료 확인 전까지 유지보수 화면을 사용한다.

```sh
docker compose stop wakapi
docker compose exec -T db sh -c 'exec pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' > "$backup_dir/pre-upgrade.dump"
docker compose exec -T db pg_restore --list < "$backup_dir/pre-upgrade.dump" > "$backup_dir/pre-upgrade.contents.txt"
docker compose up -d --no-deps wakapi
docker compose logs -f wakapi
```

백업 명령이 실패하면 새 앱을 시작하지 않는다. `WAKAPI_SKIP_MIGRATIONS`와 실패 무시 설정을 별도로 지정했다면 둘 다 `false`로 수정한다. 로그·`docker compose exec -T wakapi /app/healthcheck`·앞 단계의 SQL을 운영 DB에서도 확인한다. SQL 접속 시 DB 이름은 `wakapi_migration_check` 대신 실제 운영 DB 이름을 사용한다.

백그라운드 보정이 끝나면 기존 계정 로그인, 기간별 대시보드, 프로젝트 조회, 에디터에서 새 heartbeat 전송을 확인하고 트래픽을 연다. 쿠키 처리 변경으로 재로그인이 필요할 수 있다. 패스키는 기본 비활성화 상태다.

## 실패 시

`docker compose stop wakapi`로 재시작을 멈추고 로그를 보관한다. 마이그레이션은 전체가 하나의 트랜잭션으로 묶이지 않으므로 이미 일부 변경됐을 수 있다. 이전 앱 이미지만 다시 올리거나, 완료 표식을 수동 삭제하는 식으로 되돌리지 않는다.

안전한 복구 경로는 **업그레이드 직전 덤프를 새 복구 DB에 복원하고, 이전 이미지가 그 복구 DB를 사용하도록 전환하는 것**이다. 실패한 DB는 조사용으로 남긴다. 예를 들어 복구 DB 이름을 `wakapi_rollback`으로 만들고 위 복원 명령의 DB 이름을 그 이름으로 바꾼다. 이후 Compose override에서 `image`를 `rollback-image.txt`의 태그로, `WAKAPI_DB_NAME`을 `wakapi_rollback`으로 지정하고 `docker compose ... up -d --no-deps --no-build wakapi`로 시작한다. 실제 override와 실행 명령에는 서버의 기존 프로젝트·파일 옵션을 함께 사용한다.

이 경로로 복구하면 백업 이후에 들어온 쓰기는 복구 DB에 없으므로 점검 중 트래픽 차단을 유지한다. 기존 볼륨을 지우는 `docker compose down -v`는 실행하지 않는다.

## 검증 범위

코드의 자동 마이그레이션 경로와 PostgreSQL 구성에 맞춰 작성한 절차다. 이번 코드 병합 작업 중 운영 DB 연결, 덤프, 복원, 실제 마이그레이션은 실행하지 않았다.
