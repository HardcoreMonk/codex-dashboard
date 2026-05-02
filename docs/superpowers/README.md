# Superpowers Specs

이 디렉터리의 `specs/*.md` 문서는 구현 전후의 설계 이력이다.

현행 운영 기준은 `AGENTS.md`, `README.md`, `CLAUDE.md`, `docs/API.md`, `docs/ARCHITECTURE.md`, `docs/SCHEMA.md`, `docs/QUALITY-GATES.md`를 우선한다. 오래된 제거 대상명, legacy 비교, 전환 계획 문구는 당시 의사결정 배경으로만 읽는다.

2026-04-30 현재 기준:

- 기본 포트: `8617`
- 서비스: `codex-web-dashboard.service`
- DB: `~/.codex/dashboard.db`
- 스키마: `PRAGMA user_version=18`
- API: 애플리케이션 정의 기준 88 HTTP routes + 1 WebSocket
- 테스트: 318 pytest
