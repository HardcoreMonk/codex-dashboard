# Governance / Wiki 통합 진행 상황

작성일: 2026-04-28
현행화: 2026-05-03

## 현재 상태

`codex-dashboard`에 `codex-project-mgmt`의 프로젝트 관리 및 production wiki 운영 기능을 통합하는 1차 MVP를 구현했고, 기존 등록 project의 lifecycle redesign run을 관리 화면에서 preview/write/lint할 수 있는 lifecycle slice를 추가했다.

관리 화면에서 `codex-zone/projects.yaml`, `codex-zone/wiki/`, `codex-zone/raw/`, 최신 audit 리포트 상태를 확인하고, 신규 프로젝트를 registry에 등록한 뒤 `project-sync.sh`, `wiki-check.sh`, `zone-track.sh`를 실행할 수 있다. 또한 등록 project의 `docs/lifecycle/runs/*.json` snapshot을 조회하고 `lifecycle-redesign-start.sh`/`lifecycle-lint.sh` JSON 실행 결과를 확인할 수 있다. 실행 이력은 기존 관리자 감사 로그에 기록된다.

## 완료된 범위

### 백엔드 API

- `GET /api/governance/summary`
  - zone root, governance repo, registry, wiki, raw snapshot, audit 경로 상태 조회
  - `projects.yaml` 프로젝트 목록 파싱
  - wiki markdown 페이지 수와 주요 파일 메타데이터 제공
  - 최신 `zone-audit-latest.md` 상단 metric 제공
- `GET /api/governance/wiki/index`
  - `wiki/index.md` 본문과 파일 메타데이터 조회
- `GET /api/governance/wiki/page?path=...`
  - wiki 내부 markdown 파일 조회
  - path traversal 차단
- `GET /api/governance/audit/latest`
  - 최신 zone audit markdown 본문과 metric 조회
- `POST /api/governance/check`
  - `codex-project-mgmt/scripts/wiki-check.sh` 실행
  - 결과를 `governance_check` 감사 로그로 기록
- `POST /api/governance/sync`
  - `codex-project-mgmt/scripts/project-sync.sh` 실행
  - 결과를 `governance_sync` 감사 로그로 기록
- `POST /api/governance/track`
  - `codex-project-mgmt/scripts/zone-track.sh` 실행
  - sync/check/audit 전체 파이프라인을 실행하고 `governance_track` 감사 로그로 기록
- `POST /api/governance/projects`
  - 신규 프로젝트를 `projects.yaml`에 append
  - 프로젝트 ID 중복과 path 중복 차단
  - zone root 외부로 나가는 path 차단
  - 선택적으로 `project-sync.sh` 즉시 실행
  - 결과를 `governance_project_add` 감사 로그로 기록
- `GET /api/governance/lifecycle/runs?project_id=...`
  - 등록 project의 `docs/lifecycle/runs/*.json` snapshot 목록 조회
  - stage status, artifacts, lint summary, 파일 메타데이터 제공
- `POST /api/governance/lifecycle/preview`
  - `lifecycle-redesign-start.sh <project_id> --topic <topic> --json` 실행
  - write 없이 planned artifacts와 run summary를 반환
  - 결과를 `governance_lifecycle_preview` 감사 로그로 기록
- `POST /api/governance/lifecycle/write`
  - 등록 project만 허용하고 `confirm: true`를 요구
  - `lifecycle-redesign-start.sh <project_id> --topic <topic> --write --json` 실행
  - 생성된 run id가 유효하면 같은 run을 자동 lint
  - 결과를 `governance_lifecycle_write` 감사 로그로 기록
- `POST /api/governance/lifecycle/lint`
  - `lifecycle-lint.sh <project_id> --run <run_id> --json` 실행
  - 결과를 `governance_lifecycle_lint` 감사 로그로 기록

### 보안 및 운영 제약

- governance API는 인증 우회 목록에 추가하지 않았다.
- 스크립트 실행은 allowlist 방식으로 `wiki-check.sh`, `project-sync.sh`, `zone-track.sh`만 허용한다.
- lifecycle script 실행은 별도 allowlist 방식으로 `lifecycle-redesign-start.sh`, `lifecycle-lint.sh`만 허용한다.
- lifecycle write는 등록 project만 대상으로 하며 `confirm: true`가 없으면 거부한다.
- lifecycle write 성공 후 같은 run id를 자동 lint하고, lint 실패는 write 결과와 감사 summary에 반영한다.
- lifecycle 감사 로그 detail에는 project id, topic, run id, return code, duration, created/error/warning count, lint 상태 같은 summary만 저장하고 전체 stdout/stderr는 저장하지 않는다.
- 스크립트 출력은 API 응답에서 길이를 제한한다.
- 기본 스크립트 실행 제한 시간은 60초이며, zone tracking은 180초다.
- 실행 환경은 `ZONE_DIR`, `PROJECTS_FILE`, `WIKI_DIR`, `RAW_DIR`를 명시적으로 주입한다.
- 프로젝트 등록은 상대 경로만 허용하며 `..`, 절대 경로, 중복 ID/path를 거부한다.

### 프론트엔드

- 관리 화면에 `Governance / Wiki` 섹션을 추가했다.
- 신규 프로젝트 등록 폼을 추가했다.
- 표시 항목:
  - 등록 프로젝트 수
  - wiki 페이지 수
  - raw snapshot 파일 수
  - wiki index 파일 상태
  - 최신 audit 파일 상태와 dirty/drift/secret/untracked metric
  - 프로젝트 목록 일부
  - `wiki/index.md` 미리보기
  - 최신 audit digest 미리보기
  - `Governance / Wiki` 카드 내부 `Lifecycle Runs` 섹션
  - lifecycle project selection, topic input
  - lifecycle run list의 stage status, artifacts, lint summary
  - preview/write/lint execution result와 stdout/stderr 접기 영역
- 액션:
  - 프로젝트 등록
  - 새로고침
  - Wiki 검사
  - 프로젝트 동기화
  - Zone 추적
  - Lifecycle preview
  - Lifecycle write
  - Lifecycle lint
- 감사 로그 필터에 `governance_check`, `governance_sync`, `governance_track`, `governance_project_add`, `governance_lifecycle_preview`, `governance_lifecycle_write`, `governance_lifecycle_lint`를 추가했다.
- `index.html` cache busting 버전은 현재 `v91`이다.

## 변경 파일 (2026-05-03 lifecycle slice)

- `main.py`
- `static/app.js`
- `static/index.html`
- `tests/test_api.py`
- `tests/test_e2e_smoke.py`
- `docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md`
- `docs/API.md`
- `README.md`

## 검증 결과

통과한 검증:

```bash
ruff check main.py tests/test_api.py
npm run build
./.venv/bin/python -m pytest tests/test_api.py -q
./.venv/bin/python -m pytest tests/test_api.py -q -k 'governance'
git diff --check -- main.py static/app.js static/index.html tests/test_api.py docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md
```

결과:

- `tests/test_api.py`: 106 passed
- governance 전용 테스트: 6 passed
- 이번 변경 파일 기준 whitespace 검사 통과

실제 로컬 서버 확인:

```bash
GET  /api/governance/summary
POST /api/governance/check
GET  /api/governance/wiki/index
```

확인 결과:

- registry와 wiki 경로가 정상 조회됨
- `wiki-check ok: 3 projects, 23 wiki markdown files` 응답 확인
- `wiki/index.md` 본문 조회 확인
- 테스트 workspace에서 신규 프로젝트 등록, registry append, sync 실행, 감사 로그 기록 확인

2026-05-01 추가 검증:

```bash
env RUFF_CACHE_DIR=/tmp/ruff-cache-codex-dashboard ./.venv/bin/ruff check main.py tests/test_api.py
npm run build
git diff --check -- main.py static/app.js static/index.html tests/test_api.py docs/API.md docs/ARCHITECTURE.md docs/QUALITY-GATES.md docs/SCHEMA.md README.md CLAUDE.md docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md
```

로컬 HTTP 확인:

```bash
GET /app
GET /static/bundle.v91.js
GET /api/governance/summary
GET /api/governance/audit/latest
```

확인 결과:

- `zone-track.sh` 버튼, 최신 audit 패널, `governance_track` 감사 필터가 HTML에 포함됨
- versioned bundle이 200으로 서빙되고 `runGovernanceTrack`가 번들에 포함됨
- 최신 audit metric 파싱 확인: registry 26, dirty 9, raw drift 0, secret hits 1
- sandbox의 `fastapi.testclient.TestClient`는 최소 FastAPI 앱에서도 startup에서 hang되어 pytest route 테스트는 이번 환경에서 완료하지 못함

2026-05-03 lifecycle documentation slice 검증:

```bash
rg -n "governance/lifecycle|Lifecycle Runs|governance_lifecycle" docs/API.md docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md README.md
git diff --check -- docs/API.md docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md README.md
```

확인 범위:

- lifecycle API 문서, 진행 문서, README feature bullet에 필요한 route/action 문자열 반영
- 이번 문서 변경 파일 기준 whitespace 검사 통과
- 이번 Task 4에서는 코드/test/build를 새로 실행하지 않고 문서 범위 검증만 수행

## 알려진 잔여 사항

- repo 전체 `git diff --check`는 기존 변경분인 `docs/adr/0004-esbuild-bundle.md:3`의 trailing whitespace 때문에 실패한다. 이번 통합 변경 파일의 whitespace 문제는 없다.
- 현재 UI는 wiki index 미리보기 중심이다. 개별 wiki 페이지 탐색/편집 기능은 아직 없다.
- `projects.yaml` 전체 편집 기능은 없다. 현재는 신규 프로젝트 append만 지원한다.
- 스크립트 실행 결과는 일회성 표시와 감사 로그에만 남는다. 별도 실행 히스토리 테이블은 없다.

## 다음 단계 후보

1. wiki 페이지 목록 탐색과 개별 페이지 뷰어 추가
2. `project-sync.sh` 실행 전 변경 preview 모드 추가
3. governance 실행 결과를 별도 이력으로 저장
4. `projects.yaml` 기존 항목 수정/비활성화 기능 추가
5. Windows client metadata(`platform`, `remote_host`, `remote_path`, `sync_mode`) 입력 UI 추가
