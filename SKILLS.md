# SKILLS.md

Codex Dashboard 저장소에서 자주 쓰는 Codex / gstack 작업 흐름 요약.

프로젝트 정책과 문서 우선순위는 `AGENTS.md` 를 따른다. 이 문서는 "어떤 작업에 어떤 skill을 먼저 쓰는가"만 짧게 정리한다.

## 기본 원칙

- 디자인/브랜딩/랜딩 변경 전: `brainstorming`
- 버그 조사 전: `systematic-debugging`, `investigate`
- 기능/버그 수정 구현 전: `test-driven-development`
- 시각 QA와 폴리시: `design-review`
- 문서 동기화: `document-release`
- 커밋/푸시/배포 흐름: `ship`
- 완료 주장 전: `verification-before-completion`

## 이 저장소에서 권장되는 매핑

### 랜딩 / 공개 UI

- 랜딩 구조, 톤, CTA, 테마 UX 검토: `brainstorming`
- 기존 랜딩 업그레이드: `supanova-redesign-engine`
- 실제 시각 polish: `design-review`

### 대시보드 앱 (`/app`)

- API/프런트 계약 변화: `test-driven-development`
- 회귀 확인: `verification-before-completion`
- 실제 화면 확인이 필요할 때: `browse` 또는 `open-gstack-browser`

### 인증 / watcher / 수집 / SQLite

- hang, 재시작, 파서 오류, watcher side effect: `investigate`
- 재현 테스트 추가 후 수정: `test-driven-development`

### 문서 / 릴리스

- README / CLAUDE / docs 동기화: `document-release`
- 원격 반영까지 포함: `ship`

## 자주 쓰는 검증 명령

```bash
npm run build
./.venv/bin/python -m pytest -q tests/test_auth.py tests/test_e2e_smoke.py
./.venv/bin/ruff check .
./.venv/bin/bandit -r main.py database.py codex_parser.py codex_watcher.py codex_collector.py -s B101,B608 -q
./.venv/bin/pip-audit --strict
curl http://127.0.0.1:8617/api/auth/me
ss -ltnp | grep 8617
```

## 현재 UI 기준 메모

- 공개 진입: `/`
- 실제 앱: `/app`
- 랜딩 테마 토글: 랜딩에만 적용
- 랜딩 시그니처: 딥 다크 / 글래스 / 앰버 / 크림 / Geist / Instrument Serif / 지그재그
