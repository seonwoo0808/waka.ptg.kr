# 원본 병합 검토 (2026-11-05)

## 후속 반영
사용자 요청에 따라 아래 해결안을 실제 master의 원본 병합에 적용했다. 루트/스크립트 Go 모듈은 1.27, CI는 `1.27.x`, Docker 빌드 이미지는 `golang:1.27-alpine`으로 맞췄다. 전체 Go 테스트, Linux amd64 앱·헬스체크 빌드, Compose 구성 검사를 통과했다. 푸시 및 운영 DB 변경은 수행하지 않았다. DB 적용 절차는 [DB 업그레이드 안내](database-upgrade-20261105.md)를 참고한다.

## 비교 기준
- 포크: seonwoo0808/waka.ptg.kr master `90ddde7`
- 원본: muety/wakapi master `aba6a44` (양쪽 remote fetch 완료)
- 공통 조상: `aa954c2`
- 공통 조상 이후 원본 284개, 포크 4개 커밋. 원본 쪽 변경 215개 파일.
- 포크 고유 변경은 브랜딩, Pretendard, 배포 워크플로, Docker/Compose, 생성 자산. Go 서비스 코드 수정은 없음.

## 실제 병합 충돌과 해결
임시 clone에서 실제 merge하여 7개 파일 충돌을 재현했다.

| 파일 | 원인 | 해결 후보에 적용한 방식 |
|---|---|---|
| .github/workflows/ci.yml | 포크에서 삭제, 원본에서 수정 | 원본 CI 복원. Go 1.27, 단위/API/메일/DB 마이그레이션 검사 정의 유지 |
| .github/workflows/release.yml | 포크에서 삭제, 원본에서 수정 | 포크의 삭제 유지. 배포 정책을 임의로 원본 릴리스 방식으로 바꾸지 않음 |
| .github/workflows/docker.yml | 포크의 master push/GHCR/cosign과 원본의 릴리스/멀티아키텍처/DockerHub 변경 중첩 | 포크 트리거·GHCR·서명 유지, 원본 액션 버전 반영, version.txt 생성 복원 |
| Dockerfile | 포크의 linux/amd64 고정과 원본 TARGETOS/TARGETARCH·헬스체크 추가 중첩 | 원본 빌드 흐름 채택. 앱과 healthcheck 타깃 아키텍처 일치, Go 1.27 이미지 지정 |
| views/head.tpl.html | PENTA 메타데이터/전체 포맷 변경과 원본 favicon 교체 중첩 | PENTA 제목·소개·Pretendard 유지, 삭제된 PNG 대신 원본 새 아이콘 경로 반영 |
| static/assets/css/app.dist.css | 양쪽에서 생성된 압축 CSS를 커밋 | 자동 병합된 CSS 소스와 최종 템플릿에서 재생성 |
| static/assets/css/app.dist.css.br | 생성된 바이너리라 텍스트 병합 불가 | 새 CSS를 Brotli로 재압축 |

footer와 index, CSS 소스는 자동 병합되었으며 포크의 브랜드와 한국어 문구는 유지된다. 충돌이 없더라도 head의 옛 이미지 경로를 그대로 택하면 원본에서 삭제한 favicon 파일 때문에 링크가 깨진다.

## 원본 신규 기능과 코드 근거
| 기능 | 코드 근거 및 확인 |
|---|---|
| 패스키/WebAuthn | routes/login.go, routes/settings.go, models/webauthn.go, repositories/webauthn.go, routes/webauthn_test.go. 로컬 사용자 대상이며 기본 disable_webauthn=true |
| 프로젝트 검색·기간 유지 | views/projects.tpl.html, routes/projects.go, services/project.go, services/project_test.go. 검색 입력 및 통계 처리 경로 존재 |
| 이전/다음 기간 탐색 | static/assets/js/components/time-picker.js, views/time-picker.tpl.html. all-time 이동 비활성 처리 포함 |
| AI 코딩 통계 | models/heartbeat.go, models/compat/wakatime/v1/heartbeat.go, views/summary.tpl.html. AI/인간 라인 변경·모델 저장 및 AI coding ratio 표시 |
| AI thinking time 제외 | models/heartbeat.go 및 관련 집계 로직, testing/wakapi_api_tests의 AI thinking time 시나리오. 파일 없는 AI 세션이 코딩 시간에 포함되는 문제 보정 |
| README 통계 URL 개인화 | routes/settings.go actionUpdateReadmeStatsBaseUrl, models/user.go |
| 인증·OIDC 설정 확대 | disable_local_auth, cookie_key, oidc_insecure, config/cookie.go, config/oidc.go. 쿠키 만료/키 파생 및 OIDC 처리 변경 |
| 운영 설정 | log_format(text/json), import_hosts_whitelist, scripts/healthcheck.go 및 Docker HEALTHCHECK |

기존 포크에 이미 들어 있던 OIDC/local signup 분리와 이메일 로그인은 이번 신규 기능에 포함하지 않았다.

## 호환성 및 운영상 확인
- Go 요구 버전 1.25 → 1.27. 로컬 기본 버전은 1.25.1이었으나 자동 toolchain 다운로드 후 테스트 실행 가능했다.
- BREAKING CHANGE: API key 인증을 API 엔드포인트로 제한. 웹 summary/settings 등을 API key로 열던 연동은 수정 필요.
- routes/relay/relay.go 삭제. 구 relay 엔드포인트를 쓰던 클라이언트는 확인 필요.
- WakaTime URL 검증/redirect 제한, 공유 캐시 키 및 권한별 캐시 분리 보안 수정 포함.
- 요약/기간 집계, 사용자 timezone, historical heartbeat, relay retry 및 캐시 성능 개선 다수.
- WebAuthn·이메일/OIDC 정규화·SQLite 시간 저장·인덱스/고아 duration 정리 등 DB migration 추가. 운영 DB 백업과 복제 DB에서의 마이그레이션 검증 후 반영할 것.
- compose.yml은 포크 설정을 유지했다. 배포 환경의 기존 볼륨과 환경변수가 실제로 어떻게 주입되는지는 검사하지 않았다.

## 검증 결과와 한계
- 원본 자체 `go test ./...`: 통과.
- 충돌 해결 후보 `go test ./...`: 통과.
- 초기 제한 환경 실행의 인증/WebAuthn/DNS 관련 실패는 접근 제약을 해제한 비교 실행에서 사라졌다.
- `npm run build:all:compress`: 통과. CSS, 아이콘 및 Brotli 생성 완료.
- 미해결 Git 충돌: 0개 (임시 후보 index에 모두 해결 처리).
- diff whitespace 검사에는 원본에서 유입된 trailing whitespace 경고가 남아 있다.
- Bruno API/메일 통합 테스트, 브라우저 화면 확인, 실제 DB 마이그레이션, Docker 이미지 빌드/실행, GitHub Actions 실행은 수행하지 않았다. 전체 단위 테스트 통과가 이들 검증을 대신하지 않는다.

## 산출물
- 검토용 후보: /private/tmp/wakapi-merge-review-20261105 (codex/upstream-merge-review, merge --no-commit 상태, 충돌 해결 staged)
- 원본 기준선: /private/tmp/wakapi-upstream-baseline-20261105
- 포크 HEAD 기준 전체 병합 패치: /private/tmp/wakapi-upstream-merge-20261105.patch
- 테스트 로그: /private/tmp/wakapi-merged-tests.log, /private/tmp/wakapi-baseline-full.log
- 위 임시 산출물은 최초 검토 시점의 후보이며 영구 보관용이 아니다. 후속 실제 반영 상태는 문서 상단을 참고한다.
