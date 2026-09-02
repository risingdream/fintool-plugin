---
name: subscription-metrics
description: fintool MCP로 실측 또는 명시적 plan 구독 원장을 MRR·operating ARR·waterfall·GRR/NRR/NDR/GDR·logo/revenue cohort·ledger reconciliation으로 계산한다. MRR/ARR, new/reactivation/expansion/contraction/churn, 로고 유지율, 매출 유지율, acquisition cohort 요청에 사용한다. LTV/CAC는 unit-economics, 미래 재무는 financial-model, invoice/cash는 billing-cashflow, 데이터 기반 재무 예측 검증은 forecast-validation으로 보낸다.
---

# Subscription Metrics

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

원격 MCP의 `subscription-metrics`를 사용한다. 첫 실행 전에
`fintool_catalog {"tool":"subscription-metrics"}`로 현재 `input_contract.call_example`, flags,
typed workflow ports를 확인한다. 예시를 복사해 실제 원장 값과 정책만 바꾸고
`fintool_run`에 `tool:"subscription-metrics"`와 객체 `flags.spec`을 준다. 문자열로 이중
직렬화하지 않는다.

수치는 성공 JSON 봉투에서만 인용한다. 전체 입력·출력 계약, 공식과 workflow port는
카탈로그 응답과 성공 봉투의 `normalized_inputs`·`reconciliation`을 따른다.

## Trigger와 용어 확인

다음 요청을 이 스킬로 처리한다.

- MRR, ARR, operating ARR, monthly recurring revenue
- MRR waterfall, new·reactivation·expansion·contraction·churn
- GRR·NRR, GDR·NDR, gross/net revenue retention
- logo retention, customer/account cohort, revenue/MRR cohort
- subscription ledger replay, closing snapshot, reconciliation

NDR는 NRR, GDR는 GRR 뜻으로 쓰이는 경우가 많지만 자동 치환하지 않는다. 사용자가 준 분자·분모,
logo 단위, 신규 고객 포함 여부, expansion cap을 먼저 확인한다. ARR도 revenue·bookings·CARR인지
물은 뒤 operating ARR 요청일 때만 이 도구로 계산한다.

## 라우팅

| 사용자 의도 | route |
|---|---|
| 실측 account/component ledger의 MRR·ARR·waterfall·cohort·검산 | `subscription-metrics` |
| ARPA·gross margin·logo churn·CAC로 LTV/CAC·payback | `unit-economics` |
| 미래 손익·현금·재무제표·런웨이 | `financial-model` |
| invoice·AR·collection·settlement·bank cash | `billing-cashflow` |
| 관측 시계열의 통계적 예측·rolling-origin·coverage | `forecast-validation` |

하나의 요청에 여러 질문이 있으면 각 도구 경계를 유지한다. MRR를 invoice·recognized revenue·cash로,
NRR를 logo churn이나 LTV로 자동 변환하지 않는다.

## 필수 질문

첫 계산 전에 다음 여섯 묶음을 확인한다. 누락을 0이나 업계 관행으로 채우지 않는다.

1. **metric policy**: 포함 status, trial, discount, tax, one-time/professional service, usage,
   reactivation, future contract, source correction 정책은 무엇인가
2. **account entity**: logo 하나를 정하는 stable account ID와 dedupe 범위는 무엇인가
3. **period/timezone**: local calendar-month `[start,end)`와 IANA timezone은 무엇인가
4. **currency**: 단일 보고 통화는 무엇이며 원장 component가 모두 같은 통화인가
5. **history coverage**: prior-positive flag만 있는가, first-positive와 월별 MRR를 포함한 full
   acquisition history가 있는가
6. **closing snapshot/evidence**: 독립 마감 snapshot, ingest `as_of`, ledger version, source record와
   evidence가 있는가

실측이면 `basis:"actual"`, 명시적으로 승인된 계획 원장이면 `basis:"plan"`이다. actual을
시나리오 입력으로 자동 복사해 plan·forecast라고 부르지 않는다.

## 입력 구성

1. catalog의 `subscription-metrics/v1` example을 복사한다.
2. `metric_policy`를 전부 채운다. P0 entity는 `account`, cohort anchor는
   `first_positive_mrr`, active basis는 `positive_eligible_mrr`다.
3. opening snapshot의 account와 component를 stable ID로 적는다. component는 단일 통화,
   1·3·12개월 fixed commitment만 지원한다.
4. 같은 account·timestamp의 변경을 atomic event group으로 묶는다. 둘 이상이면 unique
   `sequence`를 명시한다.
5. 가능하면 독립 closing snapshot과 full history를 준다. 없으면 성공할 수 있으나
   `unsupported` 또는 `unverified`가 된다.
6. `source|assumption|derived|unsupported` evidence를 field path에 연결한다.

component MRR는 `recurring_amount / billing_interval_months`다. `reported_mrr`를 주면 component
합과 정확히 같아야 한다. 자동 FX·source rounding tolerance는 없다.

## 계산과 검산

### Activity waterfall

atomic group의 account MRR가 0→positive면 supplied history로 new와 reactivation을 구분한다.
positive 증가·감소·0 전환은 각각 expansion·contraction·churn이다.

```text
closing = opening + new + reactivation + expansion - contraction - churn
operating_arr = closing_mrr * 12
```

### 동일 시작 cohort

기간 시작 positive account 집합을 고정하고 기간 중 신규 account를 제외한다.

```text
logo_retention = retained_starting_accounts / starting_accounts
grr = sum(min(opening_account_mrr, closing_account_mrr)) / starting_cohort_mrr
nrr = sum(closing_account_mrr) / starting_cohort_mrr
ending_cohort_mrr = starting_cohort_mrr + cohort_expansion - cohort_contraction - cohort_churn
```

GRR은 expansion을 cap해 1을 넘지 않는다. NRR은 같은 시작 고객 expansion을 포함해 1을 넘을 수
있다. endpoint retention과 기간 중 activity를 서로 대체하지 않는다.

### Acquisition cohort

full history의 `first_positive_mrr_at` 월에 account를 한 번 배정한다. age별 logo·gross MRR·net MRR
retention을 읽고 `not_mature` row의 nullable 값을 0으로 바꾸지 않는다. full history가 없으면
cohort rows는 `unsupported`다.

### 승인 전 확인

- `reconciliation`의 component/account/total, waterfall, retention bridge delta가 0인가
- closing snapshot이 있으면 `status:"verified"`인가
- `arithmetically_reconciled`와 `fully_explained`가 각각 무엇을 뜻하는지 구분했는가
- `warnings`와 `explanations`에 history/closing/evidence 누락이 남아 있지 않은가
- `lineage`의 ledger/policy/input/result/formula hash와 `calculation_hash`가 있는가

산술이 닫혀도 closing snapshot이 없으면 `unverified`다. evidence가 없거나 unsupported면
`fully_explained:false`다. 반대로 supplied evidence가 충분해도 관측 closing이 없으면
`fully_explained:true`, `status:"unverified"`가 동시에 가능하다.

## Batch·scenario·workflow

여러 명시적 case는 `--batch`로 실행한다. case `params`는 base spec에 존재하는 숫자 leaf의 점
경로다. 관련된 component amount, reported MRR, closing snapshot, history를 함께 바꾸어 각 case가
자체적으로 reconciliation되게 만든다.

`scenario`는 가정의 타당성을 만들지 않고 명시된 range를 case로 전개할 뿐이다. actual 원장을
미래 가정으로 승격하지 않는다. 계획 민감도라면 `basis:"plan"`과 assumption evidence를 가진
별도 base를 사용한다.

typed workflow에서 MRR는 money/`currency_per_month`, operating ARR는
money/`currency_per_year`, logo·GRR·NRR는 nullable `ratio_decimal`, 원장·cohort·evidence·lineage는
structured JSON으로 연결한다. currency·period·scope가 맞지 않으면 lossy handoff를 숨기지 않는다.

## 결과 전달

최소한 다음을 함께 말한다.

- metric policy ID/version, account entity, period/timezone, currency, basis
- opening/closing MRR, operating ARR, active logo count
- new/reactivation/expansion/contraction/churn과 classification status
- 동일 시작 cohort의 logo retention, GRR, NRR와 각 분자·분모
- acquisition cohort status와 history coverage
- reconciliation status, warnings, explanations, evidence coverage
- ledger version, revision link와 `calculation_hash`

비율을 퍼센트로 표시해도 원본 decimal을 보존한다. `null`, `unsupported`, `unverified`를 0이나
실패로 축약하지 않는다.

## 금지사항

- 자동 FX, multi-currency 합산, constant-currency movement 추정
- benchmark·목표치·missing default 자동 삽입
- churn·expansion 원인, 책임, 인과관계 추정
- 세금·세법·법정 회계·공시 적정성 판단
- revenue recognition, invoice, collection, cash 계산
- NRR를 logo churn이나 LTV로 변환
- 실측 원장을 미래 가정·forecast로 자동 승격
- 검증되지 않은 statistical forecast를 계획값으로 자동 승격. 예측 검증은 `forecast-validation`
- 목표 ARR에서 필요한 movement를 역산하는 reverse planner
- unsupported/unverified/null을 0 또는 verified로 표현

P0 밖 요청은 근사하지 않는다. multi-currency/FX bucket, variable usage MRR, segment/product
attribution, weekly/daily normalization, rounded source tolerance는 P1 보류라고 밝히고 필요한
upstream data 또는 적절한 downstream 도구를 제안한다.
