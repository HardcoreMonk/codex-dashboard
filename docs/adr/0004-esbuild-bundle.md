# ADR-0004: Tailwind CDN → esbuild 번들 + Tailwind CLI

**상태**: 확정, 산출물 추적 정책은 현재 `.gitignore` 기준으로 갱신됨
**일자**: 2026-04
**결정자**: hardcoremonk
**대체**: Tailwind CDN play 모드 + 7개 개별 `<script>` 태그

## 맥락

- Tailwind CDN (~80KB JS)이 런타임에 CSS 생성 — 프로덕션 부적합
- 7개 JS 파일 개별 로드 — HTTP 요청 7회
- JS에서 동적 추가한 Tailwind 클래스가 JIT 스캔에 누락

## 결정

`build.js` (Node 스크립트)가 두 가지를 생성:
1. **bundle.js**: 소스 JS를 의존 순서대로 concat → esbuild minify
2. **tailwind.css**: `tailwindcss -i app.css -o tailwind.css --minify`

## 근거

- 7 HTTP 요청 → 1 (`bundle.js` 로컬 빌드 약 268.7KB, sourcemap 별도)
- Tailwind CSS 빌드: 사용된 유틸리티만 포함 (`tailwind.css` 로컬 빌드 약 91.7KB)
- 소스맵 포함 → 디버깅 가능
- 현재 빌드는 로컬 기준 1초 이내로 완료되어 개발 루프에 영향이 작음

## 트레이드오프

- Node.js 빌드 의존성 추가 (esbuild, tailwindcss)
- 빌드 산출물은 현재 git ignore 대상이다. 서버 배포 시 `npm run build`를 실행하거나 별도 산출물을 포함해야 한다.
- concat 방식이라 진정한 ES module이 아님 — 기존 글로벌 스크립트 API 와 `data-action` 위임 구조를 유지

## 불채택 대안

| 대안 | 불채택 이유 |
|------|-----------|
| Vite | 과도한 설정, HMR 불필요 (단일 페이지 대시보드) |
| Webpack | 설정 복잡도, 빌드 속도 느림 |
| ES module `type="module"` | 전역 함수/이벤트 위임 기반 모듈 간 계약을 import/export 계약으로 재정의해야 함 |
