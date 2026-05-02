# AGENTS.md

Codex Dashboard 프로젝트의 최상위 기준 문서.

이 저장소의 제품 정체성, 운영 기본값, 문서 우선순위, 실행 표준은 이 문서를 기준으로 한다. 세부 구현 불변식은 `CLAUDE.md`, 인터페이스 계약은 `docs/API.md`, 구조 설명은 `docs/ARCHITECTURE.md`, 데이터 계약은 `docs/SCHEMA.md`를 따른다.

## 제품 기준

- 제품명: `Codex Dashboard`
- 기본 대상: Codex CLI 세션 수집, 탐색, 복기
- 기본 포트: `8617`
- 기본 DB 경로: `~/.codex/dashboard.db`
- 기본 백업 경로: `~/.codex/dashboard-backups`
- 기본 인증 방식: `DASHBOARD_PASSWORD` 기반 쿠키 세션

새 문서와 운영 예시는 모두 Codex 기준으로 작성한다. Claude 관련 경로와 서비스는 문서와 운영 예시에서 제거한다.

## 문서 우선순위

문서 간 설명이 충돌하면 아래 순서를 따른다.

1. `AGENTS.md`
2. `CLAUDE.md`
3. `README.md`
4. `docs/API.md`
5. `docs/ARCHITECTURE.md`
6. `docs/SCHEMA.md`
7. `docs/QUALITY-GATES.md`
8. `docs/adr/*`

## 문서 책임

- `README.md`: 설치, 실행, 로그인, 운영 진입점
- `CLAUDE.md`: 코드 수정 시 지켜야 할 구현 불변식
- `docs/API.md`: HTTP/WebSocket 계약
- `docs/ARCHITECTURE.md`: 시스템 구조와 데이터 흐름
- `docs/SCHEMA.md`: SQLite 스키마와 마이그레이션
- `docs/QUALITY-GATES.md`: 머지 전 검증 기준
- `docs/adr/*`: 확정된 설계 결정 기록

공통 정책은 이 문서에 두고, 나머지 문서는 각자 자기 책임 범위만 유지한다.

## Agent Workflow Contract

- Issue tracker 규칙은 `docs/agents/issue-tracker.md`를 따른다.
- Triage label/status 매핑은 `docs/agents/triage-labels.md`를 따른다.
- Domain language와 ADR 소비 규칙은 `docs/agents/domain.md`를 따른다.
- Codex는 사용자가 명시하지 않은 issue 생성, close, label 변경, commit, push를 하지 않는다.

## 운영 원칙

- 운영 기본값은 Codex 단일 인스턴스다.
- `codex-web-dashboard.service`를 우선 설명한다.
- Codex는 `~/.codex`, `8617`, `codex-web-dashboard.service`를 기준으로 운영한다.
- 운영 환경에서는 반드시 `DASHBOARD_PASSWORD`를 설정한다.
- 기본 검증 절차는 `8617` 바인딩, `/api/auth/me`, `/api/stats` 확인으로 통일한다.

## 실행 표준

```bash
cp .env.example .env
./start.sh
```

기본 접속 주소:

```text
http://localhost:8617
```

기본 검증:

```bash
ss -ltnp | grep 8617
curl http://127.0.0.1:8617/api/auth/me
curl -i http://127.0.0.1:8617/api/stats
```

## 빌드와 테스트

```bash
npm run build
./.venv/bin/python -m pytest tests/ -v
ruff check .
```

버그 수정은 재현 테스트를 먼저 추가하고, 수정 후 관련 테스트와 회귀 범위를 확인한다.

## Plan Grilling
- `grill-me`는 원본 installer를 설치하지 않고 Codex zone의 `Plan Grilling` workflow로 사용한다.
- 신규 기능/프로젝트 설계는 `superpowers:brainstorming` 뒤, `superpowers:writing-plans` 전에 `grill-me 방식으로 검토해줘`라고 호출한다.
- 질문은 한 번에 하나만 하고, 각 질문에는 Codex의 추천 답을 함께 제시한다.
- 코드/문서로 확인 가능한 내용은 사용자에게 묻지 않고 직접 확인한다.
- `CONTEXT.md`, `CONTEXT-MAP.md`, `docs/adr/`가 있으면 용어 충돌과 ADR 후보를 함께 검토한다.
- `npx skills@latest add mattpocock/skills`, `scripts/link-skills.sh`, Claude hook installer는 실행하지 않는다.

## Lifecycle Control Plane
- 표준 lifecycle contract는 zone 상대 경로 `codex-project-mgmt/docs/codex-lifecycle-control-plane.md`를 따른다.
- 기본 순서: `intake -> superpowers:brainstorming -> grill-me -> plan-design-review -> superpowers:writing-plans -> plan-eng-review -> implement -> code-review -> release -> operate`.
- 실제 spec, grill-me 기록, plan, handoff는 해당 project root의 project-local 산출물로 둔다.
- 새 기능, behavior change, workflow contract change, multi-file change는 lightweight path를 사용하지 않는다.
- `release` 이후에는 `docs/operations/YYYY-MM-DD-<topic>-handoff.md` 또는 project-equivalent handoff로 운영 진입 상태를 기록한다.
