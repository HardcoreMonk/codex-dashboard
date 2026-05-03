---
lifecycle_run: 2026-05-03-codex-dashboard-redesign
lifecycle_stage: superpowers:brainstorming
lifecycle_status: draft
generated_by: lifecycle-redesign-start
generated_at: 2026-05-03T00:00:00
redaction_applied: false
---
# Existing Project Redesign Design: codex-dashboard

## Redesign Summary

This draft starts an existing-project redesign lifecycle from current project docs and code signals.

## Current Project Signals

- `AGENTS.md`: AGENTS.md, 제품 기준, 문서 우선순위, 문서 책임
- `CLAUDE.md`: CLAUDE.md — 코드 수정 불변식, 파일 구조, 실행·빌드·테스트, 원격 수집
- `README.md`: Codex Dashboard, 빠른 시작, 사용자 접근 검증 절차, 환경변수 (.env)
- `docs/API.md`: REST API, 자동 생성 스펙 (FastAPI), 인증, Codex 런타임 접근 검증
- `docs/ARCHITECTURE.md`: 아키텍처 가이드, 시스템 개요, 서비스 원칙, 접근 검증 체크포인트
- `docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md`: Governance / Wiki 통합 진행 상황, 현재 상태, 완료된 범위, 백엔드 API
- `docs/QUALITY-GATES.md`: 품질 게이트, Gate 1: 테스트 (필수), Gate 2: 빌드 (필수), Gate 3: 린트 (필수)
- `docs/SCHEMA.md`: 데이터베이스 스키마, 저장 원칙, file_watch_state, plan_config
- `docs/adr/0001-sqlite-single-file-db.md`: ADR-0001: SQLite 단일 파일 DB, 맥락, 결정, 근거
- `docs/adr/0002-micro-dollar-cost.md`: ADR-0002: 비용 저장 — INTEGER micro-dollars, 맥락, 결정, 근거
- `docs/adr/0003-cookie-session-auth.md`: ADR-0003: Basic Auth → 쿠키 세션 인증, 맥락, 결정, 설계
- `docs/adr/0004-esbuild-bundle.md`: ADR-0004: Tailwind CDN → esbuild 번들 + Tailwind CLI, 맥락, 결정, 근거
- `docs/adr/0005-remote-collector.md`: ADR-0005: 다중 서버 수집 — Push 방식 (collector agent), 맥락, 결정, 설계
- `docs/adr/0006-event-bus-state-accessors.md`: ADR-0006: 프론트엔드 이벤트 버스 + 상태 접근자, 맥락, 결정, 1. 이벤트 버스 (`bus`)
- `docs/adr/README.md`: Architecture Decision Records, 새 ADR 작성 시
- `docs/agents/domain.md`: Domain Docs, Read Before Work, Vocabulary, ADR
- `docs/agents/issue-tracker.md`: Issue Tracker, Backend, Rules, Publish
- `docs/agents/triage-labels.md`: Triage Labels, Category, Rules
- `docs/alert-rules.yml`: Prometheus alert rules for the Codex Dashboard., Drop this under /etc/prometheus/rules.d/ (or wherever your Prometheus, `rule_files:` scans) and `systemctl reload prometheus`., Assumes a scrape config like:

## Package And Automation Signals

- `package-lock.json`
- `package.json`
- `pyproject.toml`
- `requirements.txt`

## Redaction Summary

- Redactions: `{"args": 0, "internal_ref": 0, "local_path": 0, "secret": 0}`

## Lifecycle Contract

- Reference: `docs/codex-lifecycle-control-plane.md` or the target project's equivalent contract.

## Open Decisions

- Confirm redesign scope.
- Confirm release and operate criteria.

## Lifecycle Gate Evidence

- Stage: `superpowers:brainstorming`
- Status: `draft`
- Approved by: `not-approved`
- Evidence: Generated draft artifact. This gate is not passed yet.
