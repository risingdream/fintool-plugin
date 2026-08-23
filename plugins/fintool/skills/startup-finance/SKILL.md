---
name: startup-finance
description: fintool MCP로 스타트업 재무모델·시나리오·투자유치 자료를 만든다. 재무보고서, 피칭덱 재무, 런웨이, 자금소요, 캡테이블, LTV/CAC, bear/base/bull을 요청하면 이 스킬을 쓴다. startup-design·startup-pitch 등 스타트업 스킬 진행 중 재무 계산(Phase 7 재무, 검증 실험 판정, 피치 숫자)이 필요한 지점에서도 이 스킬을 쓴다. DCF·WACC·내재가치는 valuation 스킬로 보낸다. 숫자는 추정하지 말고 fintool 봉투만 인용한다.
---

# Startup Finance

도구는 **원격 MCP**다. `fintool_catalog`로 스키마를 보고 `fintool_run`으로 실행한다. 로컬 바이너리를 설치하지 않는다.

`fintool_catalog`는 인자 없이 부르면 도구 이름·요약 목록만 준다. 플래그가 필요하면 `{"tool": "business-plan"}`처럼 도구 하나를 지정해 받는다. 오류 메시지로 스키마를 더듬지 않는다.

이 스킬 범위: 인터뷰 → BP/유닛/시나리오/캡테이블 → `report`.  
범위 밖: DCF·WACC·기업가치(`valuation`), 포트폴리오 VaR, 채권 프라이싱.

## 원칙

1. **LLM은 숫자를 만들지 않는다.** 수치는 JSON 봉투만 인용한다.
2. **가정에 출처를 붙인다.** `source`: `창업자 인터뷰 YYYY-MM-DD`.
3. **HTML을 직접 쓰지 않는다.** `report` + `bundle.comments`.
4. **comments**는 인터뷰→모델→계산을 설명하되 새 숫자를 만들지 않는다.
5. 해시는 자르지 않는다. `business-plan`에는 해시가 없을 수 있다. 있는 것만 전문.

## 흐름

```
1. 인터뷰     → 가정 JSON (source 필수)
2. Base       → business-plan + unit-economics
3. 시나리오   → named bear/base/bull, 미조달, sample, oat
4. 캡테이블   → cap-table-simulate
5. 산출물     → bundle.json + comments → report
```

## 인터뷰 매핑

| 질문 | 플래그 |
|------|--------|
| 작년 연매출 / MRR×12 | `revenue-base` |
| 3년 성장 | `revenue-growth` (연도별 쉼표, 소수) |
| 매출 100원당 원가 | `cogs-rate` |
| 월 고정비×12 | `opex` (금액) |
| 통장 현금 | `cash-begin` |
| 조달액·프리머니 | 조달은 cash-begin에 합산. 밸류는 `price_per_share` 역산 |
| 주주·SAFE | cap-table spec |
| ARPA / churn / CAC | `arpa`, `churn-rate`, `cac` |

금액은 원(정수). 비율은 소수(`0.07` = 7%). 빠진 항목은 기본값을 제안하고 source에 `기본값 제안, 창업자 승인`.

## 호출 함정

- `fintool_run`의 `spec`은 **문자열**로 직렬화한다.
- 기본 플래그는 kebab-case. batch `params`는 **snake_case**, 스칼라만.
- **조달금은 `equity_raise` params에 넣지 않는다.** 전 연도 broadcast. 조달 케이스 `cash-begin` = 현재현금+조달액, 미조달 = 현재현금.
- named bear/base/bull은 `scenario --mode corners`가 아니다. batch cases를 직접 짠다. scenario는 sample/oat.
- `nol-carryforward: true`를 기본으로 켠다.
- unit-economics `subscription`이면 `arpa`, `arpa-period`, `gross-margin`, `churn-rate`, `churn-period`, `cac` 전부 필수. period는 같아야 한다.
- cap-table `round`는 pre-money가 아니라 `price_per_share`. SAFE가 있으면 2~3회 맞춰 수렴한다.
- `business-plan.free_cash_flow`를 `dcf`에 넣지 않는다. 그건 valuation 스킬이다.

## 산출물

봉투를 `bundle.json`으로 모은다. 형식은 `docs/examples/report/acme.bundle.json`.

```
fintool_run tool=report flags={recipe, spec, out}
```

| 레시피 | 용도 |
|--------|------|
| `finance-report` | 내부 재무 뷰. driver-trace 포함 |
| `pitch-deck` | IR. 서사·시장·Ask |
| `investor-update` | 기존 투자자 KPI·런웨이 |

`comments` 키는 부품 ID다. `pl-cash-table`, `driver-trace`, `ask-block`, `runway-track`.

조합 명세(`compose`)는 부품 ID 목록만. HTML·JSON 경로·숫자 리터럴 금지.

## startup-skill 연결

같은 플러그인의 서드파티 스킬(`startup-design`·`startup-competitors`·`startup-positioning`·`startup-pitch`)이
계산 지점에 도달하면 산출물 형식은 그 스킬을 따르되 **숫자는 전부 이 규약으로 fintool에서 낸다**.

| 트리거 | fintool 호출 |
|--------|--------------|
| Phase 7 프로젝션 (M1–12 / Y1–3) | `business-plan` (연간) / `financial-model` (월별 정밀) |
| Phase 7 민감도 ±30% | `scenario --mode oat` |
| Phase 7 conservative/base/optimistic | named bear/base/bull batch (`--mode corners` 아님) |
| Phase 7 CAC·LTV·churn | `unit-economics --model subscription` |
| Phase 7 break-even | `unit-economics --model transaction` |
| Phase 7 runway·자금소요 | 연간: `business-plan` 미조달/조달 케이스 + `montecarlo --failure-rate`. 월 단위: `financial-model` v2 `summary: true` |
| Phase 8 실험 pass/fail 판정 | `stats --mode test --test independence --method fisher-exact` |
| pitch Ask·런웨이 | `business-plan` 자금소요액 |
| pitch 희석 | `cap-table-simulate` |
| 덱·IR 산출물 | `report` (`pitch-deck` / `investor-update`) |

- `[Assumption]`/`[Estimate]` 라벨은 `assumptions[].source`에 출처로 남긴다. 고객 인터뷰 게이트 통과 전 재무는 Stage A(가정 기반)임을 명시한다.
- TAM/SAM/SOM·RICE 같은 자명한 산술은 fintool 없이 계산하고 가정 출처를 명시한다.
- 서드파티 SKILL.md는 수정하지 않는다(업데이트 시 덮어써짐). 유지보수는 이 섹션만 한다.

## 월별 런웨이: `financial-model` v2

질문이 **월 단위**(월 매출·월 고정비·월 성장률)면 연간 `business-plan` 대신 `financial-model` v2를 쓴다. 월별 현금 저점·손익분기 월·런웨이와 `calculation_hash`가 나온다. 반드시 `summary: true`로 받는다(전체 결과는 수십만 자).

최소 spec(12개월, 월 30% 성장 예시). `months`는 12~60. 이 형태에서 `months`와 숫자만 바꾼다. 배열은 길이 1(전 기간 동일) 또는 `months` 길이.

```json
{"tool":"financial-model","flags":{"summary":true,"spec":"{
  \"version\":\"financial-model/v2\",\"start_month\":\"2026-09\",\"months\":12,\"currency\":\"KRW\",
  \"opening_balance_sheet\":{\"cash\":600000000,\"accounts_receivable\":0,\"inventory\":0,\"other_current_assets\":0,\"ppe_net\":0,
    \"accounts_payable\":0,\"deferred_revenue\":0,\"other_current_liabilities\":0,\"debt\":0,\"contributed_capital\":600000000,\"retained_earnings\":0},
  \"operating\":{\"revenue\":[30000000,39000000,50700000,65910000,85683000,111387900,144804270,188245551,244719216,318134981,413575476,537648118],\"cogs\":[0],\"operating_expenses\":[80000000]},
  \"working_capital\":{\"accounts_receivable\":[0],\"inventory\":[0],\"other_current_assets\":[0],\"accounts_payable\":[0],\"deferred_revenue\":[0],\"other_current_liabilities\":[0]},
  \"ppe\":{\"capex\":[0],\"existing_depreciation_amortization\":[0],\"useful_life_months\":60},
  \"debt\":{\"draws\":[0],\"repayments\":[0],\"annual_interest_rate\":[0]},
  \"equity\":{\"raises\":[0],\"dividends\":[0]},
  \"tax\":{\"policy\":\"rate\",\"rate\":0},
  \"assumptions\":[{\"id\":\"founder_input\",\"description\":\"창업자 인터뷰 2026-08-23\"}]
}"}}
```

- **기초 현금은 `opening_balance_sheet.cash`와 `contributed_capital`에 같은 값**으로 넣는다. 빠뜨리면 런웨이가 0개월로 나온다.
- 성장률은 엔진이 전개하지 않는다. `revenue`를 `months` 길이로 직접 계산해 넣는다(3천만×1.3ⁿ).
- `revenue` 길이는 1 또는 `months`. 중간 길이는 오류다. `months`는 12~60.
- 인용은 `data.monthly[]`(월·매출·EBITDA·순이익·기말현금)·`data.summary.runway`·`data.summary.break_even`·`data.calculation_hash`. 해시는 전문.
- bear/base/bull은 `revenue` 배열만 바꿔 3번 호출한다.

## 확장

가격 실험: `pricing`. 밸류: `valuation` 스킬.
