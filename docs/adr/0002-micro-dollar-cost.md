# ADR-0002: 비용 저장 — INTEGER micro-dollars

**상태**: 확정, Codex-native 저장소에서는 legacy-compatible cost helper에 적용
**일자**: 2026-03
**결정자**: hardcoremonk

## 맥락

런타임 비용 메타데이터를 DB에 저장해야 함. 선택지: float, DECIMAL, integer micro-dollars.

2026-04-30 현재 Codex-native `codex_*` 저장소는 토큰/비용 컬럼을 별도로 저장하지 않는다. 이 ADR의 정수 비용 원칙은 legacy-compatible parser/helper와 비용 컬럼을 다시 도입하는 future migration에 적용한다.

## 결정

`cost_micro INTEGER` (1 USD = 1,000,000 micro-dollars).

## 근거

- float 누적 오차가 $0.01 단위에서 실제 발생 (수천 메시지 합산 시)
- SQLite에 DECIMAL 타입 없음
- INTEGER 연산은 정확하고 빠름
- `cost_micro * 1.0 / 1000000 AS cost_usd` 로 표시 시 변환

## 영향

- `cost_micro` 컬럼을 쓰는 경로: 정수 저장
- `cost_micro` 컬럼을 읽는 경로: `* 1.0 / 1000000` 변환 필수
- Python 코드에서 float 비용 누적 금지 — 반드시 정수 단계에서 합산
