# Governance Lifecycle Runs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add dashboard controls for previewing, creating, linting, and inspecting existing-project lifecycle redesign runs.

**Architecture:** Extend the existing Governance / Wiki API and Admin card. The dashboard calls the governance repo lifecycle shell scripts in JSON mode, returns a normalized execution envelope, records concise audit summaries, and renders project-local run snapshots without owning lifecycle state.

**Tech Stack:** FastAPI, Pydantic, SQLite admin audit log, vanilla JavaScript, static HTML, esbuild, pytest.

---

## File Structure

- Modify `main.py`: add lifecycle script constants, request models, registry target helpers, lifecycle script runner, audit summary helpers, and four `/api/governance/lifecycle/*` endpoints.
- Modify `tests/test_api.py`: extend the governance workspace fixture with lifecycle scripts and project-local run files; add API coverage for list, preview, write, lint, validation, auth, and audit.
- Modify `static/index.html`: add Lifecycle Runs controls inside the existing Governance / Wiki card and add audit filter options for lifecycle actions.
- Modify `static/app.js`: render lifecycle project selection, run list, stage grid, artifact list, and execution result blocks; wire preview/write/lint actions.
- Modify `tests/test_e2e_smoke.py`: assert lifecycle controls exist in the Admin/Governance UI and action handlers exist in source JavaScript.
- Modify `docs/API.md`: document lifecycle endpoints and constraints.
- Modify `docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md`: record lifecycle run support as the next Governance / Wiki integration scope.
- Modify `README.md`: update the Governance / Wiki feature bullet to mention lifecycle run controls.

## Task 1: Backend Lifecycle API Tests

**Files:**
- Modify: `tests/test_api.py`

- [ ] **Step 1: Extend the governance fixture with a project repo and lifecycle scripts**

Add the following lines inside `_seed_governance_workspace()` after `scripts.mkdir(parents=True)` and before writing `projects.yaml`:

```python
    demo_repo = zone / 'demo'
    run_dir = demo_repo / 'docs' / 'lifecycle' / 'runs'
    run_dir.mkdir(parents=True)
    (demo_repo / 'docs' / 'superpowers' / 'specs').mkdir(parents=True)
    (demo_repo / 'docs' / 'superpowers' / 'specs' / '2026-05-03-demo-redesign-design.md').write_text(
        '# Demo Redesign\n',
        encoding='utf-8',
    )
    (run_dir / '2026-05-03-demo-redesign.json').write_text(
        json.dumps({
            'schema_version': 1,
            'run_id': '2026-05-03-demo-redesign',
            'topic': 'demo-redesign',
            'date': '2026-05-03',
            'generated_at': '2026-05-03T00:00:00',
            'tool': 'lifecycle-redesign-start',
            'target': {'path': '.', 'project_id': 'demo'},
            'artifacts': {
                'spec': 'docs/superpowers/specs/2026-05-03-demo-redesign-design.md',
            },
            'stages': {
                'intake': 'draft',
                'superpowers:brainstorming': 'draft',
                'grill-me': 'pending',
                'plan-design-review': 'pending',
                'superpowers:writing-plans': 'pending',
                'plan-eng-review': 'pending',
                'implement': 'pending',
                'code-review': 'pending',
                'release': 'pending',
                'operate': 'pending',
            },
            'scan_summary': {},
            'redactions': {},
            'lint': {'errors': [], 'warnings': []},
            'status': 'created',
        }, indent=2) + '\n',
        encoding='utf-8',
    )
```

Also add the import at the top of `tests/test_api.py`:

```python
import json
```

- [ ] **Step 2: Add fake lifecycle scripts to the fixture**

Add these script files after `zone-track.sh` is written:

```python
    (scripts / 'lifecycle-redesign-start.sh').write_text(
        '''#!/usr/bin/env bash
set -euo pipefail
project="$1"
shift
write=false
topic=""
while [ "$#" -gt 0 ]; do
  case "$1" in
    --topic) topic="$2"; shift 2 ;;
    --write) write=true; shift ;;
    --json) shift ;;
    *) shift ;;
  esac
done
run_id="2026-05-03-${topic}"
if [ "$write" = true ]; then
  printf '{"schema_version":1,"mode":"write","target":{"project_id":"%s"},"run_id":"%s","created_artifacts":["docs/superpowers/specs/%s-design.md"],"errors":[],"warnings":[],"exit_code":0}\\n' "$project" "$run_id" "$run_id"
else
  printf '{"schema_version":1,"mode":"preview","target":{"project_id":"%s"},"run_id":"%s","planned_artifacts":{"spec":"docs/superpowers/specs/%s-design.md"},"created_artifacts":[],"errors":[],"warnings":[],"exit_code":0}\\n' "$project" "$run_id" "$run_id"
fi
''',
        encoding='utf-8',
    )
    (scripts / 'lifecycle-lint.sh').write_text(
        '''#!/usr/bin/env bash
set -euo pipefail
project="$1"
shift
run_id=""
while [ "$#" -gt 0 ]; do
  case "$1" in
    --run) run_id="$2"; shift 2 ;;
    --json) shift ;;
    *) shift ;;
  esac
done
printf '{"schema_version":1,"target":{"project_id":"%s"},"runs":[{"run_id":"%s","errors":[],"warnings":[],"stages":{"intake":"draft","superpowers:brainstorming":"draft"},"exit_code":0}],"summary":{"run_count":1,"error_count":0,"warning_count":0},"errors":[],"warnings":[],"exit_code":0}\\n' "$project" "$run_id"
''',
        encoding='utf-8',
    )
```

Add monkeypatch bindings before `return zone`:

```python
    monkeypatch.setattr(main, 'LIFECYCLE_REDESIGN_START_SCRIPT', (scripts / 'lifecycle-redesign-start.sh').resolve())
    monkeypatch.setattr(main, 'LIFECYCLE_LINT_SCRIPT', (scripts / 'lifecycle-lint.sh').resolve())
```

- [ ] **Step 3: Write tests for list, preview, write, lint, validation, and auth**

Append these tests before `test_legacy_claude_runtime_routes_are_gone`:

```python
def test_governance_lifecycle_lists_project_runs(api_client, tmp_path, monkeypatch):
    _seed_governance_workspace(tmp_path, monkeypatch)

    r = api_client.get('/api/governance/lifecycle/runs', params={'project_id': 'demo'})

    assert r.status_code == 200
    body = r.json()
    assert body['project']['id'] == 'demo'
    assert body['summary']['run_count'] == 1
    assert body['runs'][0]['run_id'] == '2026-05-03-demo-redesign'
    assert body['runs'][0]['relative_path'] == 'docs/lifecycle/runs/2026-05-03-demo-redesign.json'


def test_governance_lifecycle_preview_runs_json_script_and_audits(api_client, tmp_path, monkeypatch):
    _seed_governance_workspace(tmp_path, monkeypatch)

    r = api_client.post('/api/governance/lifecycle/preview', json={
        'project_id': 'demo',
        'topic': 'demo-redesign',
    })

    assert r.status_code == 200
    body = r.json()
    assert body['ok'] is True
    assert body['payload']['mode'] == 'preview'
    assert body['payload']['run_id'] == '2026-05-03-demo-redesign'
    assert '--write' not in body['command']

    audit = api_client.get('/api/admin/audit?action=governance_lifecycle_preview')
    assert audit.status_code == 200
    detail = audit.json()['entries'][0]['detail']
    assert detail['project_id'] == 'demo'
    assert detail['run_id'] == '2026-05-03-demo-redesign'
    assert 'stdout' not in detail
    assert 'stderr' not in detail


def test_governance_lifecycle_write_requires_confirm(api_client, tmp_path, monkeypatch):
    _seed_governance_workspace(tmp_path, monkeypatch)

    r = api_client.post('/api/governance/lifecycle/write', json={
        'project_id': 'demo',
        'topic': 'demo-redesign',
        'confirm': False,
    })

    assert r.status_code == 400
    assert r.json()['error'] == 'write confirmation required'


def test_governance_lifecycle_write_runs_lint_after_script(api_client, tmp_path, monkeypatch):
    _seed_governance_workspace(tmp_path, monkeypatch)

    r = api_client.post('/api/governance/lifecycle/write', json={
        'project_id': 'demo',
        'topic': 'demo-redesign',
        'confirm': True,
    })

    assert r.status_code == 200
    body = r.json()
    assert body['ok'] is True
    assert body['payload']['mode'] == 'write'
    assert '--write' in body['command']
    assert body['lint']['payload']['summary']['error_count'] == 0

    audit = api_client.get('/api/admin/audit?action=governance_lifecycle_write')
    assert audit.status_code == 200
    detail = audit.json()['entries'][0]['detail']
    assert detail['created_count'] == 1
    assert detail['error_count'] == 0


def test_governance_lifecycle_lint_runs_json_script_and_audits(api_client, tmp_path, monkeypatch):
    _seed_governance_workspace(tmp_path, monkeypatch)

    r = api_client.post('/api/governance/lifecycle/lint', json={
        'project_id': 'demo',
        'run_id': '2026-05-03-demo-redesign',
    })

    assert r.status_code == 200
    body = r.json()
    assert body['ok'] is True
    assert body['payload']['summary']['run_count'] == 1

    audit = api_client.get('/api/admin/audit?action=governance_lifecycle_lint')
    assert audit.status_code == 200
    assert audit.json()['entries'][0]['detail']['run_id'] == '2026-05-03-demo-redesign'


def test_governance_lifecycle_rejects_invalid_project_topic_and_run_id(api_client, tmp_path, monkeypatch):
    _seed_governance_workspace(tmp_path, monkeypatch)

    missing_project = api_client.get('/api/governance/lifecycle/runs', params={'project_id': 'missing'})
    assert missing_project.status_code == 404

    bad_topic = api_client.post('/api/governance/lifecycle/preview', json={
        'project_id': 'demo',
        'topic': 'Bad Topic',
    })
    assert bad_topic.status_code == 400

    bad_run = api_client.post('/api/governance/lifecycle/lint', json={
        'project_id': 'demo',
        'run_id': '../escape',
    })
    assert bad_run.status_code == 400


def test_governance_lifecycle_api_denied_without_login_when_password_set(auth_api_client):
    r = auth_api_client.get('/api/governance/lifecycle/runs', params={'project_id': 'demo'})

    assert r.status_code == 401
    assert r.json() == {'error': 'unauthorized'}
```

- [ ] **Step 4: Run the new tests to verify they fail**

Run:

```bash
./.venv/bin/python -m pytest tests/test_api.py -k governance_lifecycle -v
```

Expected: failures with `404 Not Found` for lifecycle routes or missing lifecycle script constants.

- [ ] **Step 5: Commit the failing tests**

```bash
git add tests/test_api.py
git commit -m "test(governance): cover lifecycle run API"
```

## Task 2: Backend Lifecycle API Implementation

**Files:**
- Modify: `main.py`
- Test: `tests/test_api.py`

- [ ] **Step 1: Add lifecycle script constants**

Add after `ZONE_TRACK_SCRIPT`:

```python
LIFECYCLE_REDESIGN_START_SCRIPT = GOVERNANCE_REPO_DIR / 'scripts' / 'lifecycle-redesign-start.sh'
LIFECYCLE_LINT_SCRIPT = GOVERNANCE_REPO_DIR / 'scripts' / 'lifecycle-lint.sh'
```

- [ ] **Step 2: Add lifecycle request models and regexes**

Add after `GovernanceProjectCreate`:

```python
class GovernanceLifecycleStartRequest(BaseModel):
    project_id: str = Field(..., min_length=1, max_length=80)
    topic: str = Field(..., min_length=1, max_length=120)
    confirm: bool = False


class GovernanceLifecycleLintRequest(BaseModel):
    project_id: str = Field(..., min_length=1, max_length=80)
    run_id: str = Field(..., min_length=12, max_length=160)


_LIFECYCLE_TOPIC_RE = re.compile(r'^[a-z0-9]+(?:-[a-z0-9]+)*$')
_LIFECYCLE_RUN_ID_RE = re.compile(r'^\d{4}-\d{2}-\d{2}-[a-z0-9]+(?:-[a-z0-9]+)*$')
```

- [ ] **Step 3: Add registered project resolution helpers**

Add after `_normalize_governance_project()`:

```python
def _governance_project_by_id(project_id: str) -> Optional[dict]:
    projects = _parse_governance_projects(PROJECTS_REGISTRY_PATH)
    for project in projects:
        if str(project.get('id')) == project_id:
            return project
    return None


def _governance_project_repo(project: dict) -> Optional[Path]:
    rel_path = str(project.get('path') or project.get('id') or '').strip()
    if not rel_path:
        return None
    repo = (ZONE_ROOT / rel_path).resolve()
    try:
        repo.relative_to(ZONE_ROOT)
    except ValueError:
        return None
    return repo


def _require_governance_project(project_id: str) -> tuple[Optional[dict], Optional[Path], Optional[JSONResponse]]:
    project = _governance_project_by_id(project_id)
    if project is None:
        return None, None, JSONResponse({'error': 'project not found'}, status_code=404)
    repo = _governance_project_repo(project)
    if repo is None:
        return project, None, JSONResponse({'error': 'project path escapes zone root'}, status_code=400)
    return project, repo, None
```

- [ ] **Step 4: Add lifecycle validation and JSON parse helpers**

Add after `_trunc_governance_output()`:

```python
def _validate_lifecycle_topic(topic: str) -> Optional[str]:
    if not _LIFECYCLE_TOPIC_RE.fullmatch(topic):
        return 'invalid lifecycle topic'
    return None


def _validate_lifecycle_run_id(run_id: str) -> Optional[str]:
    if not _LIFECYCLE_RUN_ID_RE.fullmatch(run_id):
        return 'invalid lifecycle run id'
    return None


def _parse_lifecycle_stdout(stdout: str) -> tuple[Optional[dict], Optional[str]]:
    if not stdout.strip():
        return None, 'empty JSON output'
    try:
        parsed = json.loads(stdout)
    except json.JSONDecodeError as exc:
        return None, f'invalid JSON output: {exc}'
    if not isinstance(parsed, dict):
        return None, 'JSON output is not an object'
    return parsed, None
```

- [ ] **Step 5: Add lifecycle script runner**

Add after `_run_governance_script()`:

```python
def _run_lifecycle_script(script: Path, args: list[str], *, timeout_s: int = 90) -> dict:
    allowed = {
        LIFECYCLE_REDESIGN_START_SCRIPT.resolve(),
        LIFECYCLE_LINT_SCRIPT.resolve(),
    }
    resolved = script.resolve()
    command = [str(script), *args]
    if resolved not in allowed:
        return {
            'ok': False,
            'returncode': 126,
            'script': str(script),
            'command': ' '.join(command),
            'stdout': '',
            'stderr': 'script is not allowlisted',
            'duration_ms': 0,
            'payload': None,
            'json_error': None,
        }
    if not script.exists():
        return {
            'ok': False,
            'returncode': 127,
            'script': str(script),
            'command': ' '.join(command),
            'stdout': '',
            'stderr': 'script not found',
            'duration_ms': 0,
            'payload': None,
            'json_error': None,
        }

    env = os.environ.copy()
    env.update({
        'ZONE_DIR': str(ZONE_ROOT),
        'PROJECTS_FILE': str(PROJECTS_REGISTRY_PATH),
        'WIKI_DIR': str(WIKI_DIR),
        'RAW_DIR': str(RAW_DIR),
    })
    start = time.time()
    try:
        proc = subprocess.run(
            command,
            cwd=str(GOVERNANCE_REPO_DIR),
            env=env,
            text=True,
            capture_output=True,
            timeout=timeout_s,
            check=False,
        )
        duration_ms = int((time.time() - start) * 1000)
        stdout = _trunc_governance_output(proc.stdout)
        stderr = _trunc_governance_output(proc.stderr)
        payload, json_error = _parse_lifecycle_stdout(stdout) if stdout else (None, None)
        return {
            'ok': proc.returncode == 0 and json_error is None,
            'returncode': proc.returncode,
            'script': str(script),
            'command': ' '.join(command),
            'stdout': stdout,
            'stderr': stderr,
            'duration_ms': duration_ms,
            'payload': payload,
            'json_error': json_error,
        }
    except subprocess.TimeoutExpired as e:
        duration_ms = int((time.time() - start) * 1000)
        return {
            'ok': False,
            'returncode': 124,
            'script': str(script),
            'command': ' '.join(command),
            'stdout': _trunc_governance_output(e.stdout if isinstance(e.stdout, str) else ''),
            'stderr': f'script timed out after {timeout_s} seconds',
            'duration_ms': duration_ms,
            'payload': None,
            'json_error': None,
        }
```

- [ ] **Step 6: Add lifecycle run listing and audit helpers**

Add after `_run_lifecycle_script()`:

```python
def _read_lifecycle_run_file(repo: Path, path: Path) -> dict:
    data = json.loads(path.read_text(encoding='utf-8'))
    return {
        'run_id': data.get('run_id') or path.stem,
        'status': data.get('status'),
        'date': data.get('date'),
        'topic': data.get('topic'),
        'generated_at': data.get('generated_at'),
        'stages': data.get('stages', {}),
        'artifacts': data.get('artifacts', {}),
        'lint': data.get('lint', {}),
        'relative_path': path.relative_to(repo).as_posix(),
        'file': _path_meta(path),
    }


def _list_lifecycle_runs(repo: Path) -> list[dict]:
    run_dir = repo / 'docs' / 'lifecycle' / 'runs'
    if not run_dir.exists():
        return []
    runs = []
    for path in sorted(run_dir.glob('*.json'), reverse=True):
        if path.name.startswith('.'):
            continue
        try:
            runs.append(_read_lifecycle_run_file(repo, path))
        except Exception as exc:
            runs.append({
                'run_id': path.stem,
                'relative_path': path.relative_to(repo).as_posix(),
                'error': str(exc),
                'file': _path_meta(path),
            })
    return runs


def _lifecycle_payload_summary(result: dict) -> dict:
    payload = result.get('payload') if isinstance(result.get('payload'), dict) else {}
    lint = payload.get('lint') if isinstance(payload.get('lint'), dict) else {}
    summary = payload.get('summary') if isinstance(payload.get('summary'), dict) else {}
    errors = payload.get('errors') if isinstance(payload.get('errors'), list) else lint.get('errors', [])
    warnings = payload.get('warnings') if isinstance(payload.get('warnings'), list) else lint.get('warnings', [])
    return {
        'run_id': payload.get('run_id'),
        'created_count': len(payload.get('created_artifacts') or []),
        'error_count': len(errors or []) if 'error_count' not in summary else summary.get('error_count', 0),
        'warning_count': len(warnings or []) if 'warning_count' not in summary else summary.get('warning_count', 0),
    }


def _audit_lifecycle_action(action: str, request: Optional[Request], project_id: str, topic: Optional[str], result: dict, *, run_id: Optional[str] = None) -> None:
    summary = _lifecycle_payload_summary(result)
    _audit(
        action,
        request,
        status='ok' if result.get('ok') else 'error',
        detail={
            'project_id': project_id,
            'topic': topic,
            'run_id': run_id or summary.get('run_id'),
            'returncode': result.get('returncode'),
            'duration_ms': result.get('duration_ms'),
            'created_count': summary.get('created_count', 0),
            'error_count': summary.get('error_count', 0),
            'warning_count': summary.get('warning_count', 0),
        },
    )
```

- [ ] **Step 7: Add lifecycle endpoints**

Add after `api_governance_project_create()`:

```python
@app.get("/api/governance/lifecycle/runs")
def api_governance_lifecycle_runs(project_id: str = Query(..., min_length=1, max_length=80)):
    project, repo, error = _require_governance_project(project_id)
    if error:
        return error
    runs = _list_lifecycle_runs(repo)
    return {
        'project': {
            'id': project.get('id'),
            'path': project.get('path', project.get('id')),
            'kind': project.get('kind'),
            'repo': _path_meta(repo),
        },
        'runs': runs,
        'summary': {
            'run_count': len(runs),
            'error_count': sum(1 for run in runs if run.get('error')),
        },
    }


@app.post("/api/governance/lifecycle/preview")
def api_governance_lifecycle_preview(payload: GovernanceLifecycleStartRequest, request: Request):
    project, _repo, error = _require_governance_project(payload.project_id)
    if error:
        return error
    topic_error = _validate_lifecycle_topic(payload.topic)
    if topic_error:
        return JSONResponse({'error': topic_error}, status_code=400)
    result = _run_lifecycle_script(
        LIFECYCLE_REDESIGN_START_SCRIPT,
        [payload.project_id, '--topic', payload.topic, '--json'],
    )
    _audit_lifecycle_action('governance_lifecycle_preview', request, payload.project_id, payload.topic, result)
    return result


@app.post("/api/governance/lifecycle/write")
def api_governance_lifecycle_write(payload: GovernanceLifecycleStartRequest, request: Request):
    project, _repo, error = _require_governance_project(payload.project_id)
    if error:
        return error
    if not payload.confirm:
        return JSONResponse({'error': 'write confirmation required'}, status_code=400)
    topic_error = _validate_lifecycle_topic(payload.topic)
    if topic_error:
        return JSONResponse({'error': topic_error}, status_code=400)
    result = _run_lifecycle_script(
        LIFECYCLE_REDESIGN_START_SCRIPT,
        [payload.project_id, '--topic', payload.topic, '--write', '--json'],
    )
    run_id = None
    if isinstance(result.get('payload'), dict):
        run_id = result['payload'].get('run_id')
    lint_result = None
    if run_id and _validate_lifecycle_run_id(str(run_id)) is None:
        lint_result = _run_lifecycle_script(
            LIFECYCLE_LINT_SCRIPT,
            [payload.project_id, '--run', str(run_id), '--json'],
        )
    result['lint'] = lint_result
    _audit_lifecycle_action('governance_lifecycle_write', request, payload.project_id, payload.topic, result, run_id=run_id)
    return result


@app.post("/api/governance/lifecycle/lint")
def api_governance_lifecycle_lint(payload: GovernanceLifecycleLintRequest, request: Request):
    project, _repo, error = _require_governance_project(payload.project_id)
    if error:
        return error
    run_error = _validate_lifecycle_run_id(payload.run_id)
    if run_error:
        return JSONResponse({'error': run_error}, status_code=400)
    result = _run_lifecycle_script(
        LIFECYCLE_LINT_SCRIPT,
        [payload.project_id, '--run', payload.run_id, '--json'],
    )
    _audit_lifecycle_action('governance_lifecycle_lint', request, payload.project_id, None, result, run_id=payload.run_id)
    return result
```

- [ ] **Step 8: Run focused API tests**

Run:

```bash
./.venv/bin/python -m pytest tests/test_api.py -k governance_lifecycle -v
```

Expected: all lifecycle tests pass.

- [ ] **Step 9: Run all API tests**

Run:

```bash
./.venv/bin/python -m pytest tests/test_api.py -v
```

Expected: all tests in `tests/test_api.py` pass.

- [ ] **Step 10: Commit backend implementation**

```bash
git add main.py tests/test_api.py
git commit -m "feat(governance): add lifecycle run API"
```

## Task 3: Frontend Lifecycle Controls

**Files:**
- Modify: `static/index.html`
- Modify: `static/app.js`
- Modify: `tests/test_e2e_smoke.py`

- [ ] **Step 1: Add Lifecycle Runs section markup**

In `static/index.html`, insert this block immediately after `governanceActionResult`:

```html
        <div class="mt-4 pt-4 border-t border-white/[0.05]">
          <div class="flex items-center justify-between gap-3 mb-3">
            <div>
              <div class="text-xs font-bold text-white/40">Lifecycle Runs</div>
              <div class="text-[10px] text-white/20 mt-0.5">기존 프로젝트 재설계 run 생성과 lint</div>
            </div>
            <button data-action="refreshGovernanceLifecycle" title="Lifecycle 새로고침" class="text-[10px] text-white/40 hover:text-accent spring px-2 py-1 rounded-full border border-white/[0.07]">↻</button>
          </div>
          <div class="grid grid-cols-1 md:grid-cols-[1fr_1fr_auto_auto_auto] gap-2">
            <select id="govLifecycleProject" class="bg-white/5 border border-white/[0.07] rounded-lg px-2 py-1.5 text-[11px] text-white/70 outline-none focus:border-accent/30 spring"></select>
            <input id="govLifecycleTopic" placeholder="topic-kebab-case" class="bg-white/5 border border-white/[0.07] rounded-lg px-2 py-1.5 text-[11px] text-white/70 outline-none focus:border-accent/30 spring">
            <button data-action="previewGovernanceLifecycle" class="rounded-full px-3 py-1.5 text-[10px] font-semibold bg-white/5 text-white/55 border border-white/[0.07] hover:text-white/80 spring">Preview</button>
            <button data-action="writeGovernanceLifecycle" class="rounded-full px-3 py-1.5 text-[10px] font-semibold bg-amber-500/10 text-amber-200 border border-amber-500/20 hover:scale-[1.02] active:scale-[0.98] spring">Write</button>
            <button data-action="lintGovernanceLifecycle" class="rounded-full px-3 py-1.5 text-[10px] font-semibold bg-accent/10 text-accent border border-accent/20 hover:scale-[1.02] active:scale-[0.98] spring">Lint</button>
          </div>
          <div id="governanceLifecycleRuns" class="mt-3 text-[11px]">
            <div class="text-center text-white/15 text-xs py-4">프로젝트를 선택하세요</div>
          </div>
          <div id="governanceLifecycleResult" class="mt-3 text-[10px] text-white/30"></div>
        </div>
```

- [ ] **Step 2: Add audit filter options**

In the `auditAction` select, add:

```html
            <option value="governance_lifecycle_preview">governance_lifecycle_preview</option>
            <option value="governance_lifecycle_write">governance_lifecycle_write</option>
            <option value="governance_lifecycle_lint">governance_lifecycle_lint</option>
```

- [ ] **Step 3: Add lifecycle state and project select rendering**

In `static/app.js`, add after `_governanceMetricTone()`:

```javascript
const governanceLifecycleState = {
  projects: [],
  selectedProjectId: '',
  selectedRunId: '',
  runs: [],
  lastPreview: null,
};

function _defaultLifecycleTopic(projectId) {
  return `${projectId || 'project'}-redesign`;
}

function _renderLifecycleProjectOptions(projects) {
  const select = document.getElementById('govLifecycleProject');
  if (!select) return;
  const previous = select.value || governanceLifecycleState.selectedProjectId;
  select.innerHTML = projects.map((p) => (
    `<option value="${esc(p.id || '')}">${esc(p.id || p.title || '')}</option>`
  )).join('');
  const next = projects.some((p) => p.id === previous) ? previous : (projects[0]?.id || '');
  select.value = next;
  governanceLifecycleState.selectedProjectId = next;
  const topic = document.getElementById('govLifecycleTopic');
  if (topic && !topic.value && next) topic.value = _defaultLifecycleTopic(next);
  select.onchange = () => {
    governanceLifecycleState.selectedProjectId = select.value;
    governanceLifecycleState.selectedRunId = '';
    governanceLifecycleState.lastPreview = null;
    const topicInput = document.getElementById('govLifecycleTopic');
    if (topicInput) topicInput.value = _defaultLifecycleTopic(select.value);
    loadGovernanceLifecycleRuns();
  };
}
```

- [ ] **Step 4: Call lifecycle rendering from `renderGovernanceSummary()`**

At the end of `renderGovernanceSummary(data)`, before the closing brace, add:

```javascript
  governanceLifecycleState.projects = projects;
  _renderLifecycleProjectOptions(projects);
  loadGovernanceLifecycleRuns();
```

- [ ] **Step 5: Add lifecycle run render helpers**

Add after `_renderGovernanceResult()`:

```javascript
function _lifecycleStageTone(status) {
  if (status === 'passed') return 'text-emerald-300/80 border-emerald-500/20 bg-emerald-500/10';
  if (status === 'draft') return 'text-amber-200/80 border-amber-500/20 bg-amber-500/10';
  if (status === 'blocked') return 'text-rose-300/80 border-rose-500/20 bg-rose-500/10';
  return 'text-white/35 border-white/[0.07] bg-white/[0.03]';
}

function _renderLifecycleStages(stages = {}) {
  const names = [
    'intake',
    'superpowers:brainstorming',
    'grill-me',
    'plan-design-review',
    'superpowers:writing-plans',
    'plan-eng-review',
    'implement',
    'code-review',
    'release',
    'operate',
  ];
  return `<div class="grid grid-cols-2 md:grid-cols-5 gap-1.5 mt-2">${names.map((name) => {
    const status = stages[name] || 'pending';
    return `<div class="rounded-lg border px-2 py-1 ${_lifecycleStageTone(status)}">
      <div class="text-[8px] uppercase tracking-widest truncate">${esc(name)}</div>
      <div class="text-[10px] font-bold">${esc(status)}</div>
    </div>`;
  }).join('')}</div>`;
}

function _renderLifecycleArtifacts(artifacts = {}) {
  const entries = Object.entries(artifacts || {});
  if (!entries.length) return '';
  return `<div class="mt-2 space-y-1">${entries.map(([key, value]) => (
    `<div class="text-[10px] text-white/35 truncate"><span class="text-white/20">${esc(key)}:</span> ${esc(value)}</div>`
  )).join('')}</div>`;
}

function renderGovernanceLifecycleRuns(data) {
  const el = document.getElementById('governanceLifecycleRuns');
  if (!el) return;
  const runs = data?.runs || [];
  governanceLifecycleState.runs = runs;
  if (!runs.length) {
    el.innerHTML = '<div class="text-center text-white/15 text-xs py-4">lifecycle run 없음</div>';
    return;
  }
  const selected = runs.find((run) => run.run_id === governanceLifecycleState.selectedRunId) || runs[0];
  governanceLifecycleState.selectedRunId = selected.run_id;
  const rows = runs.map((run) => `
    <button data-action="selectGovernanceLifecycleRun" data-arg0="${esc(run.run_id)}" class="w-full text-left rounded-lg px-2 py-1.5 border spring ${run.run_id === selected.run_id ? 'bg-accent/10 border-accent/20 text-accent' : 'bg-white/[0.025] border-white/[0.04] text-white/50 hover:text-white/75'}">
      <div class="font-semibold truncate">${esc(run.run_id)}</div>
      <div class="text-[9px] text-white/30 truncate">${esc(run.status || 'snapshot')} · ${esc(run.relative_path || '')}</div>
    </button>
  `).join('');
  el.innerHTML = `
    <div class="grid grid-cols-1 lg:grid-cols-[minmax(0,0.8fr)_minmax(0,1.2fr)] gap-2">
      <div class="space-y-1.5">${rows}</div>
      <div class="rounded-lg bg-white/[0.025] border border-white/[0.05] p-3">
        <div class="flex items-start justify-between gap-3">
          <div class="min-w-0">
            <div class="text-white/75 font-semibold truncate">${esc(selected.run_id)}</div>
            <div class="text-[9px] text-white/30 truncate">${esc(selected.generated_at || selected.date || 'generated unknown')}</div>
          </div>
          <div class="text-[10px] text-white/35">${esc(selected.status || 'snapshot')}</div>
        </div>
        ${_renderLifecycleStages(selected.stages || {})}
        ${_renderLifecycleArtifacts(selected.artifacts || {})}
        ${(selected.lint?.errors || []).length ? `<div class="mt-2 text-rose-300/75">${esc(selected.lint.errors.join(' · '))}</div>` : ''}
        ${(selected.lint?.warnings || []).length ? `<div class="mt-2 text-amber-200/75">${esc(selected.lint.warnings.join(' · '))}</div>` : ''}
      </div>
    </div>`;
}
```

- [ ] **Step 6: Add lifecycle API functions and actions**

Add after `renderGovernanceLifecycleRuns(data)`:

```javascript
async function loadGovernanceLifecycleRuns() {
  const projectId = document.getElementById('govLifecycleProject')?.value || governanceLifecycleState.selectedProjectId;
  const el = document.getElementById('governanceLifecycleRuns');
  if (!projectId || !el) return;
  el.innerHTML = '<div class="text-center text-white/15 text-xs py-4 dots">lifecycle runs 로딩 중</div>';
  try {
    const data = await safeFetch(`/api/governance/lifecycle/runs?project_id=${encodeURIComponent(projectId)}`);
    renderGovernanceLifecycleRuns(data);
  } catch (e) {
    renderError(el, e, loadGovernanceLifecycleRuns);
  }
}

function _governanceLifecyclePayload() {
  return {
    project_id: document.getElementById('govLifecycleProject')?.value || governanceLifecycleState.selectedProjectId,
    topic: _governanceInput('govLifecycleTopic'),
  };
}

function _renderGovernanceLifecycleResult(label, result) {
  const el = document.getElementById('governanceLifecycleResult');
  if (!el) return;
  const tone = result?.ok ? 'text-emerald-300/80' : 'text-rose-300/80';
  const payload = result?.payload || {};
  const lint = result?.lint;
  el.innerHTML = `
    <div class="${tone} font-semibold">${esc(label)} ${result?.ok ? '완료' : '실패'} · code ${esc(result?.returncode ?? '—')} · ${esc(result?.duration_ms ?? 0)}ms</div>
    ${payload.run_id ? `<div class="mt-1 text-white/40">run: <span class="text-white/65">${esc(payload.run_id)}</span></div>` : ''}
    ${payload.planned_artifacts ? _renderLifecycleArtifacts(payload.planned_artifacts) : ''}
    ${payload.created_artifacts?.length ? `<div class="mt-1 text-white/40">created: ${esc(payload.created_artifacts.length)}</div>` : ''}
    ${lint ? `<div class="mt-1 text-white/40">lint: ${esc(lint.ok ? 'ok' : 'failed')} · code ${esc(lint.returncode ?? '—')}</div>` : ''}
    <details class="mt-2 rounded-lg bg-black/20 border border-white/[0.05] p-2">
      <summary class="cursor-pointer text-white/35">실행 결과</summary>
      ${result?.stdout ? `<pre class="mt-2 max-h-44 overflow-auto text-[10px] text-white/55 whitespace-pre-wrap">${esc(result.stdout)}</pre>` : ''}
      ${result?.stderr ? `<pre class="mt-2 max-h-44 overflow-auto text-[10px] text-rose-200/70 whitespace-pre-wrap">${esc(result.stderr)}</pre>` : ''}
      ${result?.json_error ? `<div class="mt-2 text-rose-300/75">${esc(result.json_error)}</div>` : ''}
    </details>`;
}

function selectGovernanceLifecycleRun(runId) {
  governanceLifecycleState.selectedRunId = runId;
  renderGovernanceLifecycleRuns({ runs: governanceLifecycleState.runs });
}

async function previewGovernanceLifecycle() {
  const payload = _governanceLifecyclePayload();
  if (!payload.project_id || !payload.topic) {
    showToast('project와 topic을 입력하세요', { type: 'warning' });
    return;
  }
  try {
    const result = await safeFetch('/api/governance/lifecycle/preview', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });
    governanceLifecycleState.lastPreview = result;
    _renderGovernanceLifecycleResult('Lifecycle preview', result);
    loadAuditLog();
  } catch (e) {
    const el = document.getElementById('governanceLifecycleResult');
    if (el) el.textContent = `Lifecycle preview 실패: ${e.message || e}`;
    showToast(`Lifecycle preview 실패: ${e.message || e}`, { type: 'error' });
  }
}

async function writeGovernanceLifecycle() {
  const payload = { ..._governanceLifecyclePayload(), confirm: true };
  if (!payload.project_id || !payload.topic) {
    showToast('project와 topic을 입력하세요', { type: 'warning' });
    return;
  }
  if (!confirm(`${payload.project_id}에 lifecycle run을 생성합니다: ${payload.topic}`)) return;
  try {
    const result = await safeFetch('/api/governance/lifecycle/write', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });
    _renderGovernanceLifecycleResult('Lifecycle write', result);
    if (result?.payload?.run_id) governanceLifecycleState.selectedRunId = result.payload.run_id;
    loadGovernanceLifecycleRuns();
    loadAuditLog();
  } catch (e) {
    const el = document.getElementById('governanceLifecycleResult');
    if (el) el.textContent = `Lifecycle write 실패: ${e.message || e}`;
    showToast(`Lifecycle write 실패: ${e.message || e}`, { type: 'error' });
  }
}

async function lintGovernanceLifecycle() {
  const projectId = document.getElementById('govLifecycleProject')?.value || governanceLifecycleState.selectedProjectId;
  const runId = governanceLifecycleState.selectedRunId || governanceLifecycleState.runs[0]?.run_id;
  if (!projectId || !runId) {
    showToast('lint할 lifecycle run이 없습니다', { type: 'warning' });
    return;
  }
  try {
    const result = await safeFetch('/api/governance/lifecycle/lint', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ project_id: projectId, run_id: runId }),
    });
    _renderGovernanceLifecycleResult('Lifecycle lint', result);
    loadGovernanceLifecycleRuns();
    loadAuditLog();
  } catch (e) {
    const el = document.getElementById('governanceLifecycleResult');
    if (el) el.textContent = `Lifecycle lint 실패: ${e.message || e}`;
    showToast(`Lifecycle lint 실패: ${e.message || e}`, { type: 'error' });
  }
}

function refreshGovernanceLifecycle() { loadGovernanceLifecycleRuns(); }
```

- [ ] **Step 7: Add smoke assertions**

In `tests/test_e2e_smoke.py`, add:

```python
def test_governance_lifecycle_controls_are_present(e2e_client):
    html = e2e_client.get('/app').text
    app_js = Path('static/app.js').read_text()

    for marker in (
        'Lifecycle Runs',
        'govLifecycleProject',
        'govLifecycleTopic',
        'previewGovernanceLifecycle',
        'writeGovernanceLifecycle',
        'lintGovernanceLifecycle',
    ):
        assert marker in html + app_js
```

- [ ] **Step 8: Run build**

Run:

```bash
npm run build
```

Expected: build exits `0` and updates `static/bundle.js`.

- [ ] **Step 9: Run frontend smoke tests**

Run:

```bash
./.venv/bin/python -m pytest tests/test_e2e_smoke.py -v
```

Expected: smoke tests pass.

- [ ] **Step 10: Commit frontend implementation**

```bash
git add static/index.html static/app.js static/bundle.js tests/test_e2e_smoke.py
git commit -m "feat(governance): add lifecycle run controls"
```

## Task 4: Documentation

**Files:**
- Modify: `docs/API.md`
- Modify: `docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md`
- Modify: `README.md`

- [ ] **Step 1: Update API docs**

In `docs/API.md`, under the Governance / Wiki route table, add:

```markdown
| GET | `/api/governance/lifecycle/runs?project_id=<id>` | 등록 project의 `docs/lifecycle/runs/*.json` snapshot 목록 조회 |
| POST | `/api/governance/lifecycle/preview` | `lifecycle-redesign-start.sh <project_id> --topic <topic> --json` 실행. 파일 쓰기 없음. 감사 `action=governance_lifecycle_preview` |
| POST | `/api/governance/lifecycle/write` | `confirm: true`일 때 `--write --json` 실행 후 같은 run id로 lint 자동 실행. 감사 `action=governance_lifecycle_write` |
| POST | `/api/governance/lifecycle/lint` | `lifecycle-lint.sh <project_id> --run <run_id> --json` 실행. 감사 `action=governance_lifecycle_lint` |
```

After the existing project body example, add:

````markdown
Lifecycle preview/write body:

```json
{
  "project_id": "codex-dashboard",
  "topic": "codex-dashboard-redesign",
  "confirm": true
}
```

Lifecycle lint body:

```json
{
  "project_id": "codex-dashboard",
  "run_id": "2026-05-03-codex-dashboard-redesign"
}
```
````

Add these constraints to the Governance / Wiki constraints list:

```markdown
- lifecycle write는 registry 등록 project만 허용하고 `confirm: true`가 필요하다.
- lifecycle script 실행은 `lifecycle-redesign-start.sh`, `lifecycle-lint.sh`만 allowlist한다.
- lifecycle audit detail에는 stdout/stderr 전체를 저장하지 않고 요약만 기록한다.
```

- [ ] **Step 2: Update progress doc**

In `docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md`, add to completed backend/API scope:

```markdown
- lifecycle redesign run 목록, preview, write, lint API를 Governance API에 추가했다.
- write는 `confirm: true`와 registry 등록 project만 허용하고, 생성 직후 해당 run lint를 자동 실행한다.
- lifecycle preview/write/lint는 `admin_audit`에 요약 detail만 기록한다.
```

Add to frontend scope:

```markdown
- `Governance / Wiki` 카드 안에 `Lifecycle Runs` 섹션을 추가해 project 선택, topic 입력, preview/write/lint, stage 상태, 실행 결과를 표시한다.
```

- [ ] **Step 3: Update README feature bullet**

Replace the Governance / Wiki feature bullet with:

```markdown
- **Governance / Wiki**: `codex-zone/projects.yaml` 신규 project 등록, production wiki index와 최신 audit 리포트 미리보기, `project-sync.sh`/`wiki-check.sh`/`zone-track.sh` 실행, 기존 project lifecycle redesign run preview/write/lint
```

- [ ] **Step 4: Run docs grep**

Run:

```bash
rg -n "governance/lifecycle|Lifecycle Runs|governance_lifecycle" docs/API.md docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md README.md
```

Expected: API docs, progress doc, and README mention lifecycle controls.

- [ ] **Step 5: Commit docs**

```bash
git add docs/API.md docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md README.md
git commit -m "docs(governance): document lifecycle run controls"
```

## Task 5: Full Verification And Review

**Files:**
- Verify: all changed files

- [ ] **Step 1: Run API tests**

```bash
./.venv/bin/python -m pytest tests/test_api.py -v
```

Expected: all tests pass.

- [ ] **Step 2: Run smoke tests**

```bash
./.venv/bin/python -m pytest tests/test_e2e_smoke.py -v
```

Expected: all tests pass.

- [ ] **Step 3: Run build**

```bash
npm run build
```

Expected: build exits `0`.

- [ ] **Step 4: Run lint**

```bash
ruff check .
```

Expected: no lint errors.

- [ ] **Step 5: Run whitespace check**

```bash
git diff --check
```

Expected: no output and exit `0`.

- [ ] **Step 6: Confirm lifecycle route auth**

```bash
./.venv/bin/python -m pytest tests/test_api.py::test_governance_lifecycle_api_denied_without_login_when_password_set -v
```

Expected: pass with lifecycle endpoint returning `401`.

- [ ] **Step 7: Request code review**

Use `superpowers:requesting-code-review` with:

- What changed: Governance lifecycle run API/UI/docs.
- Requirements: `docs/superpowers/specs/2026-05-03-governance-lifecycle-runs-design.md`.
- Diff base: commit before Task 1.
- Diff head: current branch head.

Fix Critical and Important findings before proceeding.

- [ ] **Step 8: Final review-fix checkpoint**

Run:

```bash
git diff --quiet
```

Expected: exit `0` when review required no edits. When review fixes changed files, run:

```bash
git add main.py tests/test_api.py static/index.html static/app.js static/bundle.js tests/test_e2e_smoke.py docs/API.md docs/GOVERNANCE-WIKI-INTEGRATION-PROGRESS.md README.md
git commit -m "fix(governance): address lifecycle review findings"
```

Expected: either no diff remains after review, or a review-fix commit is created.

## Self-Review

Spec coverage:

- Backend API endpoints are covered by Tasks 1 and 2.
- Registry-only writes, topic/run validation, confirm gating, shell script JSON execution, automatic lint, and summary-only audit are covered by Task 2 tests and implementation.
- Existing Governance card UI placement, one-project run list, stage grid, artifacts, and execution output are covered by Task 3.
- Docs updates are covered by Task 4.
- Verification and review are covered by Task 5.

Completeness scan:

- The plan contains exact file paths, endpoint names, request bodies, test code, implementation snippets, commands, and expected results.
- No incomplete markers remain.

Type consistency:

- Backend models use `project_id`, `topic`, `confirm`, and `run_id`; frontend payloads and tests use the same names.
- Endpoint paths in tests, docs, and frontend match the backend path names.
- Audit action names are consistent across backend, docs, and tests.
