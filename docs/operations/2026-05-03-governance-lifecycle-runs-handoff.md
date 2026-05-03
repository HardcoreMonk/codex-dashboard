# Governance Lifecycle Runs Handoff

## Release Scope

- Change type: dashboard governance feature.
- Version change: none.
- Release branch: `main`, currently ahead of `origin/main` by 7 commits.
- Commit range: `origin/main..HEAD`.
- Released backend:
  - `GET /api/governance/lifecycle/runs?project_id=<id>`
  - `POST /api/governance/lifecycle/preview`
  - `POST /api/governance/lifecycle/write`
  - `POST /api/governance/lifecycle/lint`
- Released frontend:
  - `Governance / Wiki` card `Lifecycle Runs` controls.
  - project selector, topic input, preview/write/lint actions.
  - run list, stage grid, artifact list, execution result block, and audit filters.
- Released docs:
  - `docs/API.md`
  - `docs/ARCHITECTURE.md`
  - `docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md`
  - `README.md`

## Verification

```text
./.venv/bin/ruff check main.py tests/test_api.py tests/test_e2e_smoke.py: pass
./.venv/bin/python -m pytest tests/test_api.py -q: pass, 120 passed
./.venv/bin/python -m pytest tests/test_e2e_smoke.py -q: pass, 48 passed
npm run build: pass
git diff --check origin/main..HEAD: pass
./.venv/bin/ruff check .: pass
./.venv/bin/python -m pytest tests/ -q: pass, 334 passed, 1 warning
```

Warning observed in full pytest:

- `tests/test_websocket.py::test_ws_connect_with_cookie` reports a Starlette
  `TestClient` cookie deprecation warning. This is not introduced by the lifecycle run
  controls and does not affect release.

## Code Review

Code review findings were addressed in:

```text
35a8fcf fix(governance): address lifecycle review findings
```

The review-fix commit tightened lifecycle script output parsing, failure handling,
contracts, docs, and tests.

## Audit

Blocker definition: only issues introduced by the governance lifecycle run controls
block operate.

Known external state:

- This local branch is ahead of `origin/main` by 7 commits.
- No uncommitted code changes existed before this handoff was written.
- Generated build outputs are ignored by git.

## Blockers

- None found in fresh verification.

## Warnings

- The feature is local until the 7 commits are pushed or otherwise integrated.
- The full pytest suite is slow in this environment: 21 minutes 34 seconds.
- The dashboard remains a wrapper over `codex-project-mgmt` lifecycle scripts; lifecycle
  gate approval still belongs to project-local artifacts.

## Current Lifecycle Stage

operate entered locally after implementation, review fixes, documentation, build, lint,
focused API/UI smoke tests, and full pytest verification.

## Next Action

Choose whether to push the 7 local commits plus this handoff, create a PR, or keep the
local branch as-is for manual review.
