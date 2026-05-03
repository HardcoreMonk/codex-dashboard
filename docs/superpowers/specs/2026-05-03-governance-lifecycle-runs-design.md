# Governance Lifecycle Runs Design

## Context

`codex-project-mgmt` now provides an existing-project redesign toolkit through
`scripts/lifecycle-redesign-start.sh` and `scripts/lifecycle-lint.sh`. The first
dashboard slice should expose that toolkit from the existing `Governance / Wiki`
operator surface without making `codex-dashboard` the lifecycle state owner.

The dashboard remains a wrapper over project-local lifecycle artifacts. The canonical
state is still the target repo's markdown artifacts and `docs/lifecycle/runs/*.json`
snapshots.

## Goals

- Let an operator preview, create, lint, and inspect lifecycle redesign runs from the
  dashboard.
- Keep write access limited to registry-registered projects.
- Reuse the existing lifecycle script JSON contract rather than importing governance
  Python modules directly.
- Record dashboard lifecycle actions in the existing admin audit log with concise
  summaries.
- Fit the feature into the existing `Governance / Wiki` card without adding a new
  top-level navigation area.

## Non-Goals

- Do not create a lifecycle database in `codex-dashboard`.
- Do not support arbitrary filesystem paths or unregistered project writes.
- Do not run `project-sync.sh`, `wiki-check.sh`, or `zone-track.sh` automatically after
  lifecycle write.
- Do not mark lifecycle stages as approved from the dashboard. Gate evidence remains a
  project-local artifact concern.
- Do not redesign the Admin view layout beyond the added lifecycle section.

## Approved Decisions

- Dashboard V1 may execute the real `--write` flow.
- Write flow is `preview -> explicit write confirmation -> write -> lint -> audit`.
- Targets are registry project ids only.
- UI lives inside the existing `Governance / Wiki` card.
- The UI shows runs for one selected project at a time.
- Run creation inputs are `project_id` and `topic`; date uses the server/tool default.
- Backend invokes shell scripts in JSON mode.
- Write automatically lints only the created run.
- Script stdout, stderr, return code, and duration are shown in the UI result block.
- Audit records preview, write, and lint actions, but stores summaries only.

## Backend API

Add these authenticated endpoints under the existing governance API namespace:

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/governance/lifecycle/runs?project_id=<id>` | List project-local lifecycle run JSON files for one registered project. |
| `POST` | `/api/governance/lifecycle/preview` | Run `lifecycle-redesign-start.sh <project_id> --topic <topic> --json` without writing files. |
| `POST` | `/api/governance/lifecycle/write` | Run `lifecycle-redesign-start.sh <project_id> --topic <topic> --write --json`, then lint the created run. |
| `POST` | `/api/governance/lifecycle/lint` | Run `lifecycle-lint.sh <project_id> --run <run_id> --json`. |

`preview` and `write` request body:

```json
{
  "project_id": "codex-dashboard",
  "topic": "codex-dashboard-redesign"
}
```

`write` additionally requires:

```json
{
  "confirm": true
}
```

`lint` request body:

```json
{
  "project_id": "codex-dashboard",
  "run_id": "2026-05-03-codex-dashboard-redesign"
}
```

### Backend Validation

- `project_id` must match a project registered in `projects.yaml`.
- The resolved project path must stay under `CODEX_ZONE_ROOT`.
- `topic` must be lowercase kebab-case and match the lifecycle toolkit topic rule.
- `write` returns `400` unless `confirm` is true.
- `run_id` must match `YYYY-MM-DD-<lowercase-kebab-topic>`.
- Lifecycle scripts are allowlisted separately from the existing sync/check/track script
  allowlist.

### Script Execution

The API invokes scripts with `subprocess.run()` using the governance repo as `cwd`:

```text
scripts/lifecycle-redesign-start.sh <project_id> --topic <topic> --json
scripts/lifecycle-redesign-start.sh <project_id> --topic <topic> --write --json
scripts/lifecycle-lint.sh <project_id> --run <run_id> --json
```

The response envelope includes:

- `ok`
- `returncode`
- `duration_ms`
- `stdout`
- `stderr`
- parsed JSON payload when stdout is valid JSON
- lint result for `write`

Stdout and stderr are truncated with the existing governance output limit behavior.

### Audit

Use the existing `_audit()` path for:

- `governance_lifecycle_preview`
- `governance_lifecycle_write`
- `governance_lifecycle_lint`

Audit detail stores only summary fields:

- `project_id`
- `topic`
- `run_id`
- `returncode`
- `duration_ms`
- `created_count`
- `error_count`
- `warning_count`

Full stdout and stderr stay in the API response and are not copied into audit detail.

## Frontend UI

Extend the existing `Governance / Wiki` card with a `Lifecycle Runs` section. Do not add
a top-level navigation item.

Controls:

- project select built from `/api/governance/summary`
- topic input
- `Preview` button
- `Write` button with explicit confirmation state
- `Lint` button for a selected run

Displays:

- run list for the selected project
- selected run metadata: `run_id`, status, generated date, target project
- stage status grid using the standard lifecycle stages
- planned/created artifact paths
- errors and warnings
- collapsible execution result with return code, duration, stdout, and stderr

Interaction:

- Project selection refreshes that project's runs.
- Preview shows planned artifacts and does not enable implicit write unless the operator
  explicitly clicks write.
- Write sends `confirm: true`, then renders the write response and automatic lint result.
- Successful write refreshes the run list and selected run summary.
- Lint refreshes only the selected run result.

## Error Handling

- Invalid project, topic, or run id returns `400`.
- Unknown registry project returns `404`.
- Missing lifecycle scripts return an error response with `returncode: 127`.
- Non-allowlisted script execution returns `returncode: 126`.
- Script timeout returns `returncode: 124`.
- Invalid JSON stdout keeps the raw stdout in the response and sets parsed payload to null.
- UI shows failures in the lifecycle result block and leaves existing run data visible.

## Tests

Backend tests:

- list runs for a seeded registered project.
- preview invokes lifecycle start JSON without `--write`.
- write requires `confirm: true`.
- write invokes lifecycle start with `--write` and then lifecycle lint for the returned
  run id.
- invalid project, topic, and run id are rejected.
- lifecycle endpoints require auth when `DASHBOARD_PASSWORD` is set.
- audit entries are created for preview, write, and lint with summary-only detail.

Frontend smoke tests:

- `static/index.html` contains lifecycle controls inside the Governance card.
- bundled JavaScript contains lifecycle action handlers after build.

Verification:

- `npm run build`
- `./.venv/bin/python -m pytest tests/test_api.py -v`
- focused smoke test that already checks Admin/Governance UI markers.

## Documentation

Update these docs in the implementation slice:

- `docs/API.md`: lifecycle endpoint contract.
- `docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md`: dashboard lifecycle run support.
- `README.md`: Governance / Wiki feature bullet if needed.

## Open Issues Closed By Design

- Dashboard write access is allowed, but only for registered projects and only through the
  existing lifecycle scripts.
- Dashboard does not become source of truth; it reads project-local run snapshots.
- Cross-project sync/track remains manual.
- Audit keeps operational accountability without storing large command output.

## Spec Self-Review

- Completeness scan: no incomplete markers remain.
- Consistency check: API, UI, safety, audit, and tests all follow the same registry-only
  and script-wrapper model.
- Scope check: focused on one implementation slice and excludes lifecycle DB, broad UI
  redesign, and cross-project automation.
- Ambiguity check: write confirmation, target restrictions, script contracts, and audit
  detail limits are explicit.
