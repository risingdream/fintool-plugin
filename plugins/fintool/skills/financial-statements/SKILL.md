---
name: financial-statements
description: fintool MCP로 재무제표를 분석하고 신용위험을 계산한다. DART 재무제표에서 재무비율·듀폰 분해·EPS(기본·희석)·FCFF/FCFE·현금흐름 연결, LIFO·리스·연금·세금·외화환산 회계 정규화, Merton 부도확률·distance to default·신용스프레드, 재무지표 회귀·부도예측 로지스틱을 요청하면 이 스킬을 쓴다. 재무제표 분석, 재무비율, 부채비율, 유동비율, 이자보상배율, ROE 분해, 희석주당순이익, 잉여현금흐름, 영업현금흐름 검증, 회계 정규화, 부도확률, 신용스프레드, 신용위험, 회귀분석, 부도예측 모형 같은 한국어 요청에 반응한다. DCF·WACC·목표주가는 valuation, 펀드 성과귀속·PE 펀드지표는 fund-performance, 채권 가격·듀레이션은 fixed-income, 스타트업 런웨이·캡테이블은 startup-finance로 보낸다. 숫자는 지어내지 않고 fintool 봉투만 인용한다.
---

# Financial Statements

도구는 **원격 MCP**다. `fintool_catalog` → `fintool_run`. 로컬 바이너리를 설치하지 않는다.

범위: `fsa`(EPS·재무비율·현금흐름·회계 정규화) · `credit`(Merton 부도확률) · `regression`(OLS·로지스틱).
범위 밖: DCF·WACC·주당가치(`valuation`), 성과귀속·펀드지표(`fund-performance`), 채권 가격·듀레이션(`fixed-income`), 월별 재무모델·런웨이(`startup-finance`).

## 원칙

1. **숫자를 만들지 않는다.** 계산은 전부 fintool이 하고, 해설은 봉투 필드만 인용한다. 봉투에 없는 값을 문장에 쓰지 않는다.
2. **사용자가 준 재무제표 숫자와 내가 정한 가정을 구분해 적는다.** 특히 `credit`의 자산가치·자산변동성은 재무제표에 없는 값이라 전부 가정이다.
3. **신용등급을 추정하지 않는다.** fintool은 등급을 산출하지 않는다. 아래 「신용등급」 절을 따른다.
4. HTML을 직접 쓰지 않는다. 산출은 도구 호출과 해설이다.

## 흐름

```
1. 인터뷰   무엇을 보나 — 비율 진단? EPS? 현금흐름 품질? 부도위험? 지표 간 관계?
2. 정렬     회계기준·연결/별도·단위·기간을 맞춘다 (아래 sanity 규칙)
3. 항등식   회계등식과 손익 항등식을 먼저 맞춘다. 안 맞으면 fsa가 거절한다
4. 계산     fsa / credit / regression 호출
5. 검산     balanced · reconciled · direct_indirect_difference 확인
6. 해설     봉투 필드만 인용하고 어느 기준·어느 가정을 썼는지 밝힌다
```

## 인터뷰 — 먼저 확정할 것

한 번에 묶어서 묻고, 답이 없으면 기본값을 정하고 그 사실을 답 앞머리에 굵게 적는다.

| 질문 | 없을 때 |
|---|---|
| **회계기준이 K-IFRS인가 일반기업회계기준인가** | K-IFRS로 두고 `--accounting-standard ifrs`. 둘 다 `ifrs`에 매핑된다 — 아래 「회계기준」 절 참고 |
| **연결인가 별도인가** | DART 화면에 둘 다 있다. **묻는다.** 섞으면 비율이 통째로 틀리고 엔진은 못 잡는다 |
| 단위가 원인가 백만원인가 | 표 머리의 "(단위: 백만원)"을 확인한다. 한 호출 안에서 전부 같은 단위여야 한다 |
| 기간이 연간인가 분기인가 | 분기면 `--period-days 90`(또는 91·92). 기본 365를 그대로 두면 회전일수가 4배로 나온다 |
| 비교 시점(전기)이 있는가 | `ratios`는 기초·기말을 **둘 다** 요구한다. 전기 재무상태표가 없으면 이 모드를 못 쓴다 |
| 부도위험을 보나 | 본다면 자산의 **시장가치**와 자산변동성이 필요하다. DART엔 없다 — 아래 「credit」 절 |
| 회귀면 표본이 몇 개인가 | 계수 개수보다 많아야 하고, 실무적으로 계수당 10개 이상이 아니면 결론을 세우지 않는다 |

## 회계기준 — K-IFRS / 일반기업회계기준

`--accounting-standard`는 `ifrs`와 `us-gaap` 둘만 받는다. 한국 기준은 이렇게 매핑한다.

| 한국 기준 | 넣을 값 | 이유·주의 |
|---|---|---|
| K-IFRS (상장사·자발적 적용) | `ifrs` | IFRS 전면 채택이라 그대로 대응된다 |
| 일반기업회계기준(K-GAAP, 비상장) | `ifrs` | **`us-gaap`이 아니다.** 일반기업회계기준도 LIFO를 금지하고 리스·연금 처리가 IFRS 계열이다. `us-gaap`을 고르면 `guidance`가 ASC 조문으로 나와 오해를 부른다 |

- **`guidance`가 그대로 인용되는 자리다.** `ifrs`면 IAS 1·IAS 2·IAS 7·IAS 12·IAS 16·IAS 19·IAS 21·IAS 33·IFRS 16이 나온다. 일반기업회계기준 기업에 이 조문을 그대로 붙이면 틀린 인용이 되므로, **해설에는 "IFRS 계열 처리로 계산했고 실제 적용 기준은 일반기업회계기준"이라고 한 줄 적는다.**
- **`--adjustment inventory`(LIFO→FIFO)는 K-IFRS에도 일반기업회계기준에도 쓸 수 없다.** 둘 다 후입선출법을 금지한다. `ifrs`로 부르면 엔진이 거절한다:
  ```
  LIFO 보고값은 IFRS에서 허용되지 않으므로 inventory 조정은 accounting-standard=us-gaap이어야 합니다
  ```
  한국 기업에 이 조정을 요청받으면 **`us-gaap`으로 우회하지 말고** 적용 불가라고 답한다. 미국 상장 비교기업을 나란히 놓을 때만 쓴다.
- K-IFRS 도입 이전(2011년 이전) 재무제표를 최근 것과 비교하면 기준 자체가 다르다. 비교 표에 각주를 단다.

## 단위·기간·항등식 sanity 규칙

계산 전에 매번 확인한다. 엔진은 단위를 검증하지 못하고 그대로 계산한다.

- **모든 비율은 소수다.** 세율 22%는 `0.22`. `--tax-rate`·`--normalized-tax-rate`·`--asset-vol`·`--rate`·`--drift`·`--recovery` 전부 같다.
- **원 단위 정수는 자릿수를 다시 센다.** DART가 백만원 단위면 표의 `4,600,000`은 4.6조(`4600000000000`)다. 한 자리 어긋나면 절대금액은 10배 틀리고 비율은 멀쩡해 보여서 못 잡는다.
- **한 호출 안에서 단위를 통일한다.** `fsa`는 모든 금액 플래그가 같은 단위여야 하고, `credit`은 `--asset-value`와 `--debt`만 같으면 된다(단위가 무엇이든 상관없다).
- **`--period-days`는 연간 365가 기본이다.** 분기 재무제표를 그대로 넣으면 `days_inventory`·`days_sales_outstanding`이 4배로 나온다.
- **필수 플래그는 값이 0이어도 명시해야 한다.** 무차입 기업이라 이자비용이 0이면 `--interest-expense 0`을 **적어서** 준다. 생략하면 거절된다:
  ```
  --interest-expense를 지정해야 합니다
  ```
- **모드에 없는 플래그를 섞지 않는다.** `--solve ratios`에 `--tax-rate`를 주면 `--tax-rate는 --solve ratios에서 사용할 수 없습니다`로 거절된다. 각 모드의 플래그 집합은 아래 표에 있다.
- **회계등식을 먼저 맞춘다.** `ratios`는 `ending-assets == total-liabilities + ending-equity`가 상대오차 1e-9 안에 들어야 계산을 시작한다.
- **손익 항등식을 먼저 맞춘다.** `cashflow`는 `revenue − cogs − operating-expenses − interest-expense − tax-expense == net-income`을 강제한다. 한국 손익계산서를 그대로 옮기면 거의 항상 걸린다 — 아래 「cashflow」 절에 처리법이 있다.

## 재무비율 — `fsa --solve ratios`

DART 재무상태표·손익계산서·현금흐름표에서 24개 플래그를 채운다. **전부 필수**다.

```json
{"tool":"fsa","flags":{"solve":"ratios","accounting-standard":"ifrs","period-days":365,
 "revenue":3000000000000,"cogs":1900000000000,"operating-income":420000000000,"net-income":310000000000,
 "beginning-assets":4200000000000,"ending-assets":4600000000000,
 "beginning-equity":2400000000000,"ending-equity":2650000000000,"total-liabilities":1950000000000,
 "beginning-inventory":380000000000,"ending-inventory":420000000000,
 "beginning-receivables":520000000000,"ending-receivables":560000000000,
 "current-assets":1800000000000,"current-liabilities":900000000000,
 "cash":300000000000,"marketable-securities":120000000000,
 "total-debt":800000000000,"interest-expense":45000000000,"operating-cash-flow":520000000000,
 "nopat":330000000000,"beginning-invested-capital":3000000000000,"ending-invested-capital":3200000000000}}
```

- 인용 위치를 헷갈리지 않는다:
  - 회계등식: `data.balance_equation.{assets, liabilities, equity, difference, balanced}`
  - 구성비: `data.common_size_income.*`, `data.common_size_balance.*`
  - 활동성: `data.activity.{inventory_turnover, days_inventory, receivables_turnover, days_sales_outstanding, total_asset_turnover}`
  - 유동성: `data.liquidity.{current_ratio, quick_ratio, cash_ratio}`
  - 안정성: `data.solvency.{debt_to_assets, debt_to_equity, financial_leverage, interest_coverage, operating_cash_to_debt}`
  - 수익성: `data.profitability.{gross_margin, operating_margin, net_margin, return_on_assets, return_on_equity, return_on_invested_capital}`
  - 듀폰: `data.dupont.{net_margin, asset_turnover, financial_leverage, return_on_equity, reported_roe, difference}`
- **모든 지표는 `{value, formula}` 객체다.** 숫자는 `.value`에 있다. 분모가 0이면 `value: null`과 `reason`이 온다(예: 무차입 기업의 `interest_coverage` → `"interest expense가 0입니다"`). **null은 오류가 아니고, 해설에 `reason`을 그대로 옮긴다.**
- 비율은 **소수로 나온다.** `debt_to_equity` 0.30은 30%지 30배가 아니다.
- **⚠️ `debt_to_equity`는 한국식 "부채비율"이 아니다.** 엔진 식은 `total_debt / ending_equity`, 즉 **이자부 차입금**/자본이다. 한국 관행의 부채비율은 **총부채**/자본이고 엔진이 산출하지 않는다. 사용자가 "부채비율"을 물으면 `data.common_size_balance.total_liabilities.value ÷ data.common_size_balance.equity.value`로 직접 낸 뒤 **두 정의를 나란히 표에 적는다.** 둘을 같은 이름으로 부르면 수치가 통째로 달라진다(실측 예: 차입금 기준 30.19% vs 총부채 기준 73.6%).
- **검산**: `data.dupont.difference.value == 0`이어야 듀폰 분해가 보고 ROE와 일치한다. 해설에 한 줄로 넣는다.
- 회전율·수익률의 분모는 **기초·기말 단순평균**이다(`data.assumptions`에 명시). 분기마다 자산이 크게 변한 기업이면 이 근사가 거칠다고 적는다.

### 회계등식이 안 맞을 때

DART 표를 백만원 단위로 반올림해 옮기면 자산 ≠ 부채+자본이 되기 쉽다. 실측: 4.6조에서 5,000원 차이만 나도 거절된다.

```
회계등식이 맞지 않습니다: assets=4.600000005e+12, liabilities+equity=4.6e+12, difference=5000
```

두 가지 중 하나를 고른다. **어느 쪽을 했는지 반드시 밝힌다.**

1. **원본을 다시 본다** — 비지배지분을 자본에서 빠뜨렸는지가 가장 흔하다. `ending-equity`는 **자본총계**(지배 + 비지배)여야 `ending-assets`와 맞는다.
2. **`--tolerance`를 완화한다** — 반올림 차이가 확실하면 `--tolerance 1e-6`으로 준다. 통과하면 `balanced: true`가 되지만 `difference`에 잔차가 그대로 남으므로 **그 잔차를 해설에 인용한다.**

## 현금흐름·FCFF/FCFE — `fsa --solve cashflow`

27개 플래그가 **전부 필수**다. 직접법·간접법을 동시에 계산하고 둘을 대조한다.

```json
{"tool":"fsa","flags":{"solve":"cashflow","accounting-standard":"ifrs",
 "revenue":3000000000000,"cogs":1900000000000,"operating-expenses":680000000000,
 "interest-expense":45000000000,"tax-expense":65000000000,"net-income":310000000000,
 "noncash-charges":180000000000,
 "change-receivables":40000000000,"change-inventory":40000000000,"change-payables":25000000000,
 "change-other-operating-assets":5000000000,"change-other-operating-liabilities":10000000000,
 "change-interest-payable":2000000000,"change-tax-payable":3000000000,"tax-rate":0.22,
 "capital-expenditures":250000000000,"asset-sale-proceeds":15000000000,
 "debt-issued":100000000000,"debt-repaid":60000000000,"dividends":80000000000,"total-debt":800000000000,
 "beginning-cash":300000000000,"ending-cash":475000000000,
 "investing-cash-flow":-235000000000,"financing-cash-flow":-40000000000,"foreign-exchange-effect":5000000000}}
```

- **부호 규약**: `--capital-expenditures`·`--asset-sale-proceeds`·`--debt-issued`·`--debt-repaid`·`--dividends`·`--cogs`·`--interest-expense`·`--tax-expense`는 **전부 양수**로 준다. 반대로 `--investing-cash-flow`·`--financing-cash-flow`·`--foreign-exchange-effect`는 현금흐름표 그대로 **유출이면 음수**다. 한 호출 안에서 두 규약이 공존한다.
- `change-*`는 **기말 − 기초**다. 자산 증가는 양수(현금 사용), 부채 증가도 양수(현금 원천). 엔진이 방향을 알아서 처리한다.
- 인용: `data.direct.{customer_receipts, supplier_payments, operating_payments, interest_paid, taxes_paid, operating_cash_flow}`, `data.indirect.{net_income, noncash_charges, working_capital_investment, other_payable_adjustments, operating_cash_flow}`, `data.{fcff, fcfe, net_capital_investment, net_borrowing}`, 커버리지는 `data.coverage.{interest_coverage, debt_service_coverage, capital_expenditure_coverage, cashflow_adequacy, operating_cash_to_debt}`(전부 `{value, formula}`).
- **검산 두 가지를 매번 한다.**
  - `data.direct_indirect_difference == 0` — 직접법과 간접법 CFO가 같아야 한다.
  - `data.cash_reconciliation.reconciled == true` — 기초현금 + CFO + CFI + CFF + 환율효과 = 보고 기말현금.
- **현금 연결 실패는 오류가 아니라 경고다.** `ok: true`로 결과가 나오고 `data.warnings[]`와 `reconciled: false`가 붙는다. 실측:
  ```
  기말 현금이 연결되지 않습니다: 계산=4.75e+11, 보고=4e+11, difference=7.5e+10
  ```
  **`reconciled: false`인 결과의 FCFF·FCFE를 결론으로 쓰지 않는다.** 차이 금액을 그대로 인용하고 어느 항목이 빠졌는지 되짚는다.
- `fcff_formula`는 `operating_cash_flow + interest_expense * (1 - tax_rate) - net_capital_investment`다. **발생주의 이자비용을 쓰지 지급이자(`interest_paid`)를 쓰지 않는다.** 미지급이자 변동이 큰 기업이면 이 차이를 각주로 남긴다.

### ⚠️ 손익 항등식이 한국 손익계산서를 거절한다

`cashflow`는 직접법 변환을 위해 다음을 강제한다:

```
revenue − cogs − operating-expenses − interest-expense − tax-expense == net-income
```

기타수익·기타비용·금융수익·지분법손익·중단영업손익이 있는 실제 손익계산서를 그대로 옮기면 반드시 걸린다:

```
직접법 변환용 손익계산서가 net-income과 연결되지 않습니다: 입력=2.5e+11, revenue-cogs-operating-expenses-interest-expense-tax-expense=3.1e+11
```

**처리법**: 잔여 항목(기타수익 − 기타비용 − 지분법손익 등)을 `--operating-expenses`에 순액으로 합쳐 항등식을 맞춘다. 기타수익이 크면 `operating-expenses`가 줄고, 심하면 음수가 될 수도 있다. **이렇게 밀어넣은 금액을 해설에 명시하고, 그만큼 `data.direct`의 `operating_payments`가 실제 영업지급액이 아니라고 적는다.** 순이익을 임의로 바꿔 맞추면 안 된다.

## EPS — `fsa --solve eps`

필수는 `--net-income`·`--weighted-average-shares` 둘뿐이고 나머지는 선택이다.

```json
{"tool":"fsa","flags":{"solve":"eps","accounting-standard":"ifrs",
 "net-income":310000000000,"preferred-dividends":5000000000,"weighted-average-shares":60000000,
 "option-shares":"2000000","option-exercise-prices":"45000","average-market-price":70000,
 "convertible-numerator-adjustments":"12000000000","convertible-incremental-shares":"9000000"}}
```

- `--net-income`은 **보통주 귀속 순이익**이다. 연결이면 비지배지분을 뺀 지배주주지분 순이익을 쓴다.
- `--weighted-average-shares`는 **주식분할·무상증자가 소급 반영된** 가중평균 주식수다(`data.assumptions`에 명시). DART 주석의 가중평균유통보통주식수를 그대로 쓴다.
- 옵션·전환증권은 **배열 플래그**다. 문자열로 쉼표 구분하거나 `@파일.json`으로 준다. `option-shares`와 `option-exercise-prices`, `convertible-numerator-adjustments`와 `convertible-incremental-shares`는 **길이가 같아야 한다.**
- 인용: `data.{earnings_available, basic_shares, basic_eps, diluted_numerator, diluted_shares, diluted_eps, dilution_per_share}`, 증권별은 `data.instruments[].{kind, series, incremental_shares, incremental_eps, included, reason, eps_after_inclusion}`.
- **엔진이 반희석 판정을 자동으로 한다.** 증분 EPS가 낮은 순서로 포함하고, EPS를 올리는 증권은 제외한다. 제외되면 `included: false`와 `reason`이 붙는다:
  - `antidilutive: 포함하면 EPS가 증가하거나 손실주당액의 절대값이 감소합니다`
  - `out_of_the_money: exercise price가 average market price 이상입니다`
- **제외된 증권을 해설에서 빠뜨리지 않는다.** "전환사채 900만 주는 반희석이라 희석EPS에서 제외됐다"까지 써야 사용자가 잠재 희석 규모를 안다.
- 옵션은 treasury-stock 방식이라 `--average-market-price`가 없으면 증분 주식수를 낼 수 없다. 기간 평균 주가를 구할 수 없으면 옵션을 빼고 계산하고 그 사실을 적는다.

## 회계 정규화 — `fsa --solve adjust`

비교가능성 조정이다. **재무제표 재작성이 아니고 감사의견을 대체하지 않는다**(`data.assumptions`에 명시). 결과는 `data.<kind>` 아래에 들어가고 `data.<kind>.formulas`에 계산식이 그대로 온다.

| `--adjustment` | 쓰는 곳 | 필수 플래그 | 인용 |
|---|---|---|---|
| `inventory` | LIFO→FIFO. **`us-gaap` 전용** | `reported-inventory`, `reported-cogs`, `reported-net-income`, `reported-equity`, `beginning-lifo-reserve`, `ending-lifo-reserve`, `tax-rate` | `data.inventory.{fifo_inventory, fifo_cogs, fifo_net_income, fifo_equity, change_in_lifo_reserve, tax_effect}` |
| `long-lived-asset` | 비용 처리 원가를 분석상 자본화 | `reported-assets`, `reported-equity`, `reported-operating-income`, `reported-net-income`, `current-expensed-cost`, `normalization-amortization`, `unamortized-capitalized-cost`, `tax-rate` | `data.long_lived_asset.{normalized_assets, normalized_equity, normalized_operating_income, normalized_net_income, pretax_income_adjustment}` |
| `lease` | 부외·단일 임차료를 ROU자산·리스부채로 재구성 | `reported-assets`, `reported-liabilities`, `reported-operating-income`, `reported-net-income`, `right-of-use-asset`, `lease-liability`, `lease-expense`, `lease-depreciation`, `lease-interest`, `tax-rate` | `data.lease.{normalized_assets, normalized_liabilities, normalized_operating_income, normalized_net_income, pretax_income_adjustment}` |
| `pension` | 근무원가는 영업, 이자−기대수익은 금융으로 재분류 | `projected-benefit-obligation`, `plan-assets`, `reported-operating-income`, `reported-interest-expense`, `reported-pension-expense`, `service-cost`, `pension-interest-cost`, `expected-return`, `pension-contribution` | `data.pension.{funded_status, net_pension_asset, net_pension_liability, normalized_operating_income, normalized_finance_cost, operating_contribution, financing_contribution}` |
| `tax` | 지속가능 정상세율로 순이익 재계산 | `pretax-income`, `reported-tax-expense`, `cash-taxes-paid`, `normalized-tax-rate`, `reported-net-income`, `deferred-tax-assets`, `deferred-tax-liabilities` | `data.tax.{effective_tax_rate, cash_tax_rate, deferred_tax_component, normalized_tax_expense, normalized_net_income, net_deferred_tax_liability}` |
| `foreign-currency` | current-rate / temporal 환산 | `translation-method`, `monetary-assets`, `nonmonetary-assets`, `monetary-liabilities`, `nonmonetary-liabilities`, `foreign-revenue`, `foreign-operating-expenses`, `historical-rate-expenses`, `translated-equity-before-adjustment`, `current-rate`, `average-rate`, `historical-rate` | `data.foreign_currency.{method, translated_assets, translated_liabilities, translated_revenue, translated_expenses, translated_net_income, translation_adjustment, oci_translation_adjustment, profit_loss_remeasurement}` |

- **`inventory`는 한국 기업에 쓰지 않는다.** K-IFRS도 일반기업회계기준도 LIFO를 금지한다. 요청받으면 적용 불가라고 답하고, 미국 비교기업이 있을 때만 그쪽에 쓴다.
- `lease`는 IFRS 16 도입 이후 K-IFRS 기업엔 이미 재무상태표에 올라와 있다. **이 조정이 필요한 건 (a) 일반기업회계기준 적용 비상장사의 운용리스, (b) IFRS 16 이전 재무제표와의 비교**다. 어느 경우인지 밝힌다.
- `pension`은 확정급여형(DB)에만 의미가 있다. 확정기여형(DC)이면 조정할 게 없다.
- `foreign-currency`의 환율은 **표시통화 / 현지통화 1단위**다(`rate_convention`에 명시). USD 재무제표를 원화로 환산하면 `--current-rate 1380`처럼 준다. 반대로 넣으면 조용히 1/1,900,000 규모로 틀린다.
- `--tax-rate`는 **한계세율**이다. 한국 법인세는 지방소득세 포함 실효 약 22~26% 구간이니 어느 값을 왜 썼는지 적는다.

## 신용위험 — `credit`

Merton 구조모형이다. 주식을 자산가치에 대한 유럽형 콜옵션(행사가 = 부채 액면)으로 보고, 부도확률 = N(−d₂)를 낸다.

```json
{"tool":"credit","flags":{"asset-value":4600000000000,"debt":1950000000000,
 "asset-vol":0.28,"maturity":1,"rate":0.032,"drift":0.07,"recovery":0.4}}
```

- 인용: `data.{d1, d2, distance_to_default, default_probability, equity_value, debt_value, risk_free_debt_value, debt_yield, credit_spread, credit_spread_bp, leverage, expected_loss, loss_given_default, deterministic}`, 측도별은 `data.measures[].{measure, drift, distance_to_default, default_probability}`.
- **최상위 `default_probability`는 위험중립(risk-neutral) 값이다.** 실측도(physical) 값은 `--drift`를 줬을 때만 `data.measures`에 두 번째 항목으로 생긴다. **어느 측도를 인용하는지 문장에 밝힌다.** 위험중립 PD는 스프레드 가격결정용이고, "이 회사가 부도날 확률"에 가까운 건 실측도 쪽이다.
- **`credit_spread`는 소수, `credit_spread_bp`는 bp다.** 둘을 섞어 부르지 않는다.
- `distance_to_default`는 d₂와 같은 값이다. 자산이 부채보다 작으면 음수가 되고, 그때도 계산은 된다.
- `--recovery`를 주면 `expected_loss = default_probability × loss_given_default × risk_free_debt_value` 근사가 붙는다. **분모가 부채 액면이 아니라 액면의 현재가치**이므로 액면 기준 손실률로 다시 나누지 않는다. 회수율을 상수로 본 근사고, 구조모형이 내재하는 회수율(부도 시점의 자산가치)과는 다른 값이다.
- **`data.assumptions`를 그대로 해설에 옮긴다.** 자산가치가 GBM을 따르고 변동성이 상수·점프 없음, 부채를 만기 T의 무이표채 하나로 뭉뚱그림, **부도는 만기에만 판정**(만기 전 부도를 안 잡아 단기 PD가 과소평가), 자산가치·자산변동성은 관측값이 아닌 입력값.
- `data.warnings[]`가 있으면 반드시 인용한다. 실측 임계값:
  | 조건 | 경고 |
  |---|---|
  | `asset-vol > 1.0` | `자산변동성 1.2는 연율 소수 기준으로 매우 큽니다(0.3 = 30%). 주가변동성을 그대로 넣지 않았는지 확인하세요` |
  | `default_probability > 0.5` | `이 영역에서는 Merton 모형의 단일 만기·만기시점 부도 가정이 특히 무리한 단순화입니다` |
  | `maturity > 10` | `부채를 무이표채 하나로 뭉뚱그린 가정이 특히 거칠어지는 구간입니다` |
  | `asset-value < debt` | `지금 청산하면 상환 불능인 상태이고, 부도확률이 높게 나오는 것은 그 결과입니다` |
  | `asset-vol == 0` | `확률이 아니라 확정 판정입니다. distance_to_default는 정의되지 않아 null입니다` |

### ⚠️ 입력이 DART에 없다

`--asset-value`(자산의 **시장가치**)와 `--asset-vol`(**자산**수익률 변동성)은 관측되지 않는 값이다. 엔진은 **주식 시가총액에서 역산하는 기능을 지원하지 않는다.** 그러니 이 둘은 전부 사용자 가정이고, 결과의 신뢰도는 가정의 출처가 결정한다.

재무제표만 있을 때 쓸 수 있는 근사와 그 한계를 **반드시 함께 적는다**:

| 입력 | 근사 | 한계 |
|---|---|---|
| `--asset-value` | 시가총액 + 총부채 장부가 | 부채를 장부가로 본다. 부실기업일수록 부채 시장가가 장부가보다 낮아 자산가치가 과대평가된다 |
| `--asset-value` (비상장) | 자산총계 장부가 | 시장가가 아니다. **PD가 통째로 무의미해질 수 있다.** 비상장사엔 이 도구를 권하지 않는다 |
| `--asset-vol` | 주가변동성 × (시가총액 / 자산가치) | 1차 근사다. 정식으로는 주식가치·주식변동성에서 연립방정식으로 역산해야 하고 그건 이 도구가 안 한다 |
| `--debt` | 유동부채 + 비유동부채 절반 (KMV 관행) | 부도점을 어떻게 잡느냐가 PD를 크게 흔든다. 무엇을 썼는지 적는다 |
| `--maturity` | 1 (연간 PD) | 부채 만기 구조를 무시한다 |

**세 가지 이상을 가정으로 채웠으면 PD를 단독 결론으로 쓰지 않는다.** 대신 가정을 바꿔가며 `credit`을 여러 번 부르고 PD 범위를 보고한다.

### 신용등급은 추정하지 않는다

**fintool에 신용등급 산출·매핑 기능이 없다.** 나오는 것은 PD·distance to default·credit spread뿐이다.

- **PD를 등급으로 옮기지 않는다.** "PD 0.12%면 A0급" 같은 문장을 쓰지 않는다. 등급은 신용평가사가 재무·사업·계열지원 요인을 종합해 부여하는 판단이고, Merton PD 한 숫자로 환산되지 않는다.
- **엔진 스스로 밝히는 편향이 있다.** 실제 기업은 만기가 제각각인 부채를 여럿 갖고, 부도는 만기 전에도 일어나며, 자산가치는 점프한다. 그래서 **Merton PD는 등급별 실측 부도율보다 대체로 낮게 나온다.** 등급 비교를 요청받으면 이 편향을 먼저 말한다.
- **할 수 있는 것**: 같은 방식으로 계산한 동종업계 기업들의 `distance_to_default`를 **서로** 비교하는 것. 절대 수준이 아니라 순위로 읽고, 모든 기업에 같은 가정(부도점 정의·만기·변동성 산출법)을 썼다는 사실을 명시한다.
- 사용자가 실제 등급을 알고 있으면 그것을 받아 적고, `credit_spread_bp`가 시장에서 관측되는 해당 등급 스프레드와 얼마나 벌어지는지를 논한다. **시장 스프레드를 지어내지 않는다** — 사용자가 주거나, 모르면 모른다고 한다.

## 회귀 — `regression`

`--model`은 **필수**다. 빼면 `--model을 지정해야 합니다 (ols|logistic)`로 거절된다.

### OLS — 재무지표 간 관계

```json
{"tool":"regression","flags":{"model":"ols","confidence":0.95,"input":{
 "response":[0.121,0.098,0.135,0.088,0.142,0.115,0.076,0.129,0.104,0.091,0.138,0.112],
 "predictors":[
  {"name":"debt_ratio","numeric":[0.72,0.95,0.61,1.12,0.55,0.80,1.31,0.66,0.88,1.02,0.58,0.77]},
  {"name":"sales_growth","numeric":[0.08,0.03,0.11,0.01,0.14,0.06,-0.02,0.10,0.05,0.02,0.12,0.07]}],
 "predict":[{"label":"기준케이스","numeric":{"debt_ratio":0.85,"sales_growth":0.06}}]}}}
```

- 인용: `data.coefficients[].{name, estimate, standard_error, statistic, p_value, confidence_interval, significant}`, `data.fit.{sample_size, parameters, degrees_freedom, r_squared, adjusted_r_squared, residual_standard_error}`, `data.anova.{regression, residual, total}`, `data.predictions[].{label, predicted, standard_error, confidence_interval, prediction_interval}`, 진단은 `data.diagnostics.{breusch_pagan, durbin_watson, vif}`, 잔차는 `data.residuals[]`·`data.fitted_values[]`.
- **신뢰구간과 예측구간을 섞지 않는다.** `confidence_interval`은 **평균**의 구간, `prediction_interval`은 **개별 관측치**의 구간이고 후자가 항상 넓다. 특정 기업의 다음 기 값을 말할 때는 `prediction_interval`이다.
- 범주형은 `{"name":..., "categorical":[...], "reference":"기준범주"}`로 준다. 계수 이름이 `standard[k-gaap]`처럼 `이름[수준]` 형태로 나오고 **기준 범주 대비 차이**로 읽는다.

### 로지스틱 — 부도예측

```json
{"tool":"regression","flags":{"model":"logistic","max-iterations":100,"tolerance":1e-10,"input":{
 "response":[0,0,1,0,1,0,0,1,0,0,1,1,0,1,0,0,1,0,0,1],
 "predictors":[
  {"name":"interest_coverage","numeric":[8.2,5.1,1.2,6.4,0.8,4.9,7.3,1.5,5.6,9.1,0.6,1.9,6.8,4.2,1.1,7.7,3.4,2.1,5.9,6.2]},
  {"name":"debt_to_equity","numeric":[0.6,0.9,2.4,0.7,3.1,1.0,0.5,2.2,0.8,0.4,3.5,2.0,0.6,1.1,2.8,0.5,1.6,2.5,0.9,1.3]}],
 "predict":[{"label":"심사대상","numeric":{"interest_coverage":2.0,"debt_to_equity":1.8}}]}}}
```

- `response`는 **0 또는 1만** 받는다. 아니면 `logistic response의 3번째 값은 0 또는 1이어야 합니다 (받은 값: 2)`.
- 인용: `data.coefficients[].{estimate, p_value, odds_ratio, confidence_interval, significant}`, `data.fit.{sample_size, log_likelihood, null_log_likelihood, likelihood_ratio, likelihood_ratio_p_value, mcfadden_r_squared, aic, bic, accuracy}`, `data.{converged, iterations, probabilities, classes}`, `data.predictions[].{label, probability, class, confidence_interval}`, 다중공선성은 **`data.vif[]`(최상위)**.
- **`data.converged`를 먼저 본다.** `false`면 계수를 인용하지 않는다.
- **`data.classes`와 `predictions[].class`는 임계값 0.5 고정이다.** 부도 표본이 드문 데이터(부도율 5% 같은)에서 0.5는 거의 아무도 부도로 분류하지 않아 `accuracy`가 높게 나온다. **`accuracy`를 모형 성능으로 인용하지 않는다.** `probability` 자체를 쓰고 임계값은 용도에 맞게 사용자와 정한다.
- `odds_ratio`는 `exp(estimate)`다. "이자보상배율이 1 늘면 부도 오즈가 0.45배" 식으로 읽고, **확률이 0.45배가 아니라는 점**을 짚는다.
- `mcfadden_r_squared`는 OLS의 R²와 다르다. 0.2~0.4면 좋은 적합으로 본다.

### ⚠️ 회귀 실패 세 가지

실측한 오류 메시지와 원인이다.

| 증상 | 메시지 | 원인·처리 |
|---|---|---|
| 로지스틱 발산 | `로지스틱 Newton step 계산 실패: matrix singular or near-singular with condition number 2.2539e+16` | **완전분리(complete separation)**. 설명변수 하나가 부도·정상을 완벽히 가른다. IRLS가 계수를 ±∞로 밀어 수렴하지 않는다. 해당 변수를 빼거나 표본을 늘린다. **`--max-iterations`를 키워도 해결되지 않는다.** |
| 표본 부족 | `관측치가 2개인데 추정할 계수가 3개입니다. 관측치는 계수보다 많아야 합니다` | 범주형은 수준 수 − 1개의 계수를 만든다. 계수 수를 다시 센다 |
| 길이 불일치 | `predictor "x"의 관측치가 3개입니다. response 4개와 같아야 합니다` | 결측 연도를 지우다 한쪽만 지운 경우가 흔하다 |

- **VIF를 항상 본다.** 재무비율은 서로 강하게 얽혀 있다. 실측 예: 부채비율과 매출성장률만 넣은 12개 표본에서 VIF 28.2(`high: true`). VIF가 10을 넘으면 개별 계수의 부호·크기를 해석하지 않는다.
- **표본이 짧으면 결론을 세우지 않는다.** `data.fit.sample_size`를 항상 함께 인용하고, 계수당 10개 미만이면 한계로 적는다. R² 0.99에 계수 p값 0.31이 같이 나오는 상황은 적합이 좋은 게 아니라 표본이 부족한 것이다.
- **회귀는 인과가 아니다.** "부채비율이 ROE를 낮춘다"가 아니라 "표본에서 음의 상관이 관측된다"로 쓴다.

## DART에서 직접 받는다 — `--from`

**사용자가 숫자를 불러주기를 기다리기 전에 DART에서 받아온다.** 받은 값은 봉투 `sources[]`에
provider·기준일·공시 원문 URL과 함께 실린다. 사람이 화면을 보고 옮겨 적는 단계가 빠지면
자릿수 실수와 "이 숫자 어디서 왔더라"가 함께 사라진다.

### ref 하나가 여러 입력을 채운다

```
dart:financials:<6자리코드>:<연도>:<FY|Q1|H1|Q3>[:<CFS|OFS>]
```

| `fsa` 입력 | DART 표준계정 | 비고 |
|---|---|---|
| `revenue` · `cogs` · `operating-income` · `net-income` | 매출액·매출원가·영업이익·당기순이익 | 손익은 기간값이라 기초가 없다 |
| `ending-assets` / `beginning-assets` | 자산총계 (당기/전기) | `ratios`가 둘 다 요구한다 |
| `ending-equity` / `beginning-equity` | 자본총계 (당기/전기) | |
| `ending-inventory` / `beginning-inventory` | 재고자산 (당기/전기) | |
| `current-assets` · `current-liabilities` · `total-liabilities` · `cash` | 유동자산·유동부채·부채총계·현금및현금성자산 | 시점값 |
| `ending-receivables` / `beginning-receivables` | 매출채권및기타유동채권 (당기/전기) | 표준계정이 매출채권과 기타유동채권을 한 줄로 낸다 |
| `total-debt` | 차입금 계정 **합산** | 단기·유동성장기·장기차입금 + 사채 + 리스부채. 내역은 `data.debt_components` |

`total-debt`는 표준계정 한 줄이 아니라 합산이고 **리스부채를 포함한다**(IFRS 16 이후 이자부부채).
다른 정의를 쓰려면 `fetch`로 `debt_components`를 먼저 보고 `--total-debt`를 직접 준다.
차입금 계정을 하나도 못 찾으면 아예 채우지 않는다 — 무차입 기업과 구분되어야 하기 때문이다.

**여기 없는 값은 `--from`으로 채워지지 않는다.** 표준계정으로 안전하게 집을 수 없는 것들이고,
비슷한 이름을 조용히 집어 틀린 값을 넣느니 묻는 편이 낫다. **여전히 사용자에게 묻는다:**

- `ratios`: `marketable-securities` · `interest-expense` · `operating-cash-flow` · `nopat` ·
  `beginning-invested-capital` · `ending-invested-capital`
- `eps`: `weighted-average-shares` · 지배주주지분 순이익(있으면) · 잠재주식
- `cashflow`: 운전자본 증감 항목 전부 · `tax-rate` · CAPEX · 배당 · 환율효과
- `credit`: 자산의 **시장가치**와 자산변동성 — 「credit」 절 참고. DART에 없다

**`fsa --solve ratios`는 `--solve`를 포함한 필수 플래그 24개를 요구하고 `--from`이 채우는 재무 입력은 17개다.** 나머지 6개를 받기 전에는
`ratios` 전체를 부르지 못한다. `--from`으로 대부분을 채웠다고 인터뷰를 건너뛰지 않는다.

**단, 필요한 비율만 고르면 그 입력만 있으면 된다.** `--metrics`로 지표 그룹을 고른다.

```bash
# 영업이익률 하나가 필요하면 dart:financials 하나로 끝난다
fintool fsa --solve ratios --metrics margin --from dart:financials:012510:2025:FY:CFS
```

| 그룹 | 산출 | `dart:financials` 하나로 되는가 |
|---|---|:---:|
| `balance` | 회계등식 검증 | ✅ |
| `common-size` | common-size 손익·재무상태 | ✅ |
| `margin` | 매출총이익률·영업이익률·순이익률 | ✅ |
| `activity` | 재고회전·재고일수·매출채권회전·DSO·총자산회전 | ✅ |
| `dupont` | 3단계 DuPont | ✅ |
| `liquidity` | 유동·당좌·현금비율 | ❌ `marketable-securities` |
| `leverage` | 부채비율·재무레버리지·이자보상·영업현금/부채 | ❌ `interest-expense` `operating-cash-flow` |
| `return` | ROA·ROE·ROIC | ❌ `nopat` `invested-capital` |

고르지 않은 그룹은 결과에서 빠지고 `omitted_metrics`에 이유가 남는다. **비율 하나를 못 낸다고
원값 두 개를 병치해 우회하지 않는다** — 그 하나를 `--metrics`로 골라 도구에게 계산시킨다.

### 주식 수는 별도 ref다

```
dart:shares:<코드>:<연도>[:<FY|Q1|H1|Q3>][:<outstanding|issued>]
```

재무제표 API에는 주식 수가 없다. `eps`의 분모나 주당 지표가 필요하면 이 ref로 받는다.
기본은 **보통주 유통주식수**(`outstanding`, 자기주식 제외)이고, 발행주식총수가 필요하면
`:issued`를 붙인다. IAS 33이 EPS 분모로 요구하는 것도, 배수가 서는 것도 유통주식수 쪽이다.
`basis` 필드가 어느 쪽을 냈는지 말한다 — `by_kind`와 대조할 수 있다.

**단, `weighted-average-shares`는 이 값이 아니다.** 가중평균은 기중 증자·자기주식 취득을 반영하므로
기말 유통주식수와 다르다. 유상증자나 자사주 거래가 있었으면 사업보고서 주석의 가중평균 수를 쓴다.
변동이 없었다면 `dart:shares` 값을 그대로 쓰고 그 사실을 답에 적는다.

### 비상장사는 재무수치가 나오지 않는다

`dart:search`는 외부감사 대상 비상장사도 찾아주고(`listed: false`, 8자리 `corp_code`),
`dart:filings`도 그 고유번호로 감사보고서 목록을 준다. 그러나 **`dart:financials`는 정기보고서를
제출하는 회사만** 다루므로 비상장사 재무수치는 이 경로로 나오지 않는다. 그 실패는 조회 조건 실수가
아니라 API의 경계이고, 오류 메시지가 다음 수(`dart:filings`로 감사보고서 원문 확인)를 말해준다.

### 호출 형태 — 플래그만 쓴다

원격 MCP는 위치 인자를 받지 않는다. 좌변마다 같은 ref를 반복해 주면 봉투는 **출처 하나로 합치고**
`fields`에 채운 입력 이름을 모두 적는다.

EPS는 DART가 분자를, 사용자가 분모를 준다.

```json
{"tool":"fsa","flags":{"solve":"eps",
  "from":["net-income=dart:financials:005930:2025:FY:CFS"],
  "weighted-average-shares":5919637922}}
```

`ratios`는 자동 취득분과 사용자 입력을 한 호출에 섞는다. 같은 ref를 좌변마다 반복한다.

```json
{"tool":"fsa","flags":{"solve":"ratios","accounting-standard":"ifrs",
  "from":["revenue=dart:financials:005930:2025:FY:CFS",
          "cogs=dart:financials:005930:2025:FY:CFS",
          "operating-income=dart:financials:005930:2025:FY:CFS",
          "net-income=dart:financials:005930:2025:FY:CFS",
          "beginning-assets=dart:financials:005930:2025:FY:CFS",
          "ending-assets=dart:financials:005930:2025:FY:CFS",
          "beginning-equity=dart:financials:005930:2025:FY:CFS",
          "ending-equity=dart:financials:005930:2025:FY:CFS",
          "beginning-inventory=dart:financials:005930:2025:FY:CFS",
          "ending-inventory=dart:financials:005930:2025:FY:CFS",
          "current-assets=dart:financials:005930:2025:FY:CFS",
          "current-liabilities=dart:financials:005930:2025:FY:CFS",
          "total-liabilities=dart:financials:005930:2025:FY:CFS",
          "cash=dart:financials:005930:2025:FY:CFS",
          "beginning-receivables=dart:financials:005930:2025:FY:CFS",
          "ending-receivables=dart:financials:005930:2025:FY:CFS",
          "total-debt=dart:financials:005930:2025:FY:CFS"],
  "marketable-securities":0,"interest-expense":0,"operating-cash-flow":0,"nopat":0,
  "beginning-invested-capital":0,"ending-invested-capital":0}}
```

**위 `0`들은 자리를 보여주는 것이지 기본값이 아니다.** 사용자에게 받은 실제 숫자로 바꾼다.
`--from`과 같은 입력에 값 플래그를 함께 주면 `conflicting_input`으로 거절한다 — 둘은 겹치지 않게 나눈다.

회사명은 계산 ref에 넣지 않는다. **동명이의로 조용히 틀린 회사를 집는 실패**가 정확히 이 도구가 피하려는 것이다.
코드를 모르면 먼저 확인한다.

```json
{"tool":"fetch","flags":{"ref":"dart:search:삼성전자"}}
```

### 인터뷰 항목을 ref가 대신 대답한다

| 인터뷰 질문 | ref에서 어떻게 정해지나 |
|---|---|
| 연결인가 별도인가 | ref 끝의 `:CFS`(연결) / `:OFS`(별도). **생략하면 연결이다.** 묻지 말고 ref에 명시해 답에도 그대로 적는다 |
| 기간이 연간인가 분기인가 | ref의 `FY`·`Q1`·`H1`·`Q3`. 분기·반기면 어댑터가 **누적 금액**을 쓰고 그 사실을 `sources[].note`에 남긴다 |
| 단위가 원인가 백만원인가 | DART 원본이 **원 단위**다. 사용자가 부른 숫자와 섞지 않는다 |
| 회계기준 | ref가 답하지 않는다. 「회계기준」 절대로 여전히 확인한다 |

- 비교 대상이 있으면 **두 회사 모두 같은 `:CFS`/`:OFS`와 같은 기간 코드로 받는다.** ref가 다르면 비교가 무의미하다.
- **2015년 이전은 제공되지 않는다.** 그 연도 요청은 `source_not_found`로 돌아온다.
- 정정보고서로 값이 바뀔 수 있다. 어떤 보고서를 근거로 계산했는지는 `sources[].note`의 **접수번호**가 말하고,
  `sources[].url`이 그 공시 원문 뷰어로 바로 열린다. 값이 이상하면 그 링크를 사용자에게 준다.
- DART의 `as_of`는 날짜가 아니라 **보고 기간**이다(`2025-FY`). 금리·시세의 `as_of`(`2026-08-21`)와 형태가 다르므로
  둘을 한 표에 나란히 적을 때 헷갈리지 않게 라벨을 붙인다.

### 출처를 어떻게 인용하나

`sources[]`의 `provider`·`as_of`·`url`을 **답과 리포트에 함께 적는다.**

> 매출액·자산총계 등 8개 항목 — 금융감독원 Open DART, 삼성전자(005930) 2025 사업보고서 연결(CFS)
> (`dart.fss.or.kr/dsaf001/main.do?rcpNo=…`). 공시정보의 정확성·완전성은 보장되지 않는다(DART 이용약관 제23조).

- **`sources[].fields`에 이름이 있는 입력만 자동 취득이다.** 거기 없는 입력 — `total-debt`·`interest-expense`·
  `credit`의 자산가치·자산변동성 — 은 사용자 입력이거나 내가 정한 가정이고, 그 셋을 한 줄에 섞어 적지 않는다.
  「원칙」 2번이 요구하는 구분이 봉투에 기계적으로 남는 자리가 여기다.
- ECOS 금리를 함께 쓰면(`credit`의 무위험이자율 등) ECOS 이용약관 제7조②에 따라 **"한국은행 ECOS 제공"을 함께 표시한다.**
- `sources[].cached`가 `true`면 파일 캐시에서 온 값이다. `as_of`는 그대로 유효하다.

### 키가 없거나 소스가 막혔을 때

`missing_credential`(키 없음) · `quota_exceeded`(DART 일 20,000건 초과) · `source_not_found`(그 연도·코드 없음)로
분류되어 돌아온다. **계산을 포기하지 않는다.**

1. 오류 메시지에 조회 화면 URL과 대체할 수기 플래그가 들어 있다. 사용자에게 그대로 전달하고
   아래 「DART 재무제표 인터뷰」 순서로 받아 적는다.
2. **자동 취득과 수기 입력은 `data`가 바이트 단위로 같다.** `sources`는 `calculation_hash` 밖이라 해시도 같다.
   수기로 갔다고 결과가 달라지지 않는다.
3. 그 실행의 봉투에는 `sources`가 없다. 그러면 해설에도 출처를 적지 않는다 — 봉투가 모르는 것을 문장이 아는 척하지 않는다.
4. `quota_exceeded`는 재시도하지 않는다. 같은 호출이 같은 이유로 다시 실패한다.

## DART 재무제표 인터뷰 — 받아 적는 순서

`--from`으로 채우지 못하는 값과 키가 없을 때의 폴백 경로다. 숫자를 받을 때 이 순서로 확인한다.
각 항목은 사용자가 주는 값이고, 내가 채우면 가정이라고 표시한다.

```
1. 회사·기간·기준   회사명, 사업연도, 연결/별도, K-IFRS/일반기업회계기준, 단위(원/백만원)
2. 재무상태표       자산총계·부채총계·자본총계(기초·기말) → 회계등식 먼저 검산
                    유동자산·유동부채·현금·단기금융상품·재고·매출채권·차입금
3. 손익계산서       매출·매출원가·판관비·영업이익·금융비용·법인세비용·당기순이익
                    (지배주주지분 순이익을 따로 받는다 — EPS 분자)
4. 현금흐름표       영업/투자/재무 현금흐름, 감가상각비, 운전자본 증감, CAPEX, 배당, 환율효과
5. 주식             가중평균유통보통주식수, 잠재주식(옵션·CB·BW), 기간 평균주가
6. 검산             ratios의 balanced, cashflow의 reconciled와 direct_indirect_difference
7. 각주             회계기준 매핑 근거, 내가 채운 가정, null로 나온 지표의 reason
```

- **비교 대상이 있으면 같은 기준으로 뽑았는지 먼저 확인한다.** A사는 연결, B사는 별도로 계산해 나란히 놓으면 부채비율 비교가 무의미하다.
- **분기 데이터는 누적인지 당기인지 확인한다.** DART 분기보고서의 손익은 누적 표기가 흔하다. 재무상태표는 시점값이라 섞이면 회전율이 틀린다.
- 재무비율에 **업종 평균이나 등급 기준선을 지어내지 않는다.** 사용자가 주면 쓰고, 없으면 자기 시계열(전기 대비)로만 논한다.

## 비율 해석 코멘트 규약

숫자를 낸 뒤 붙이는 해설은 이 규칙을 따른다.

1. **봉투 값 → 그 값이 뜻하는 것 → 한계** 순서로 세 문장 안에 쓴다.
2. **좋다·나쁘다를 단정하지 않는다.** 기준선 없이 "유동비율 2.0은 양호하다"고 쓰지 않는다. "유동비율 2.0(`data.liquidity.current_ratio.value`)은 유동부채의 2배를 유동자산으로 덮는다"까지가 사실이고, 양호 여부는 업종 기준선이 있어야 판단된다.
3. **비율이 움직인 이유는 분자·분모를 각각 짚는다.** ROE가 올랐으면 듀폰 세 요소(`data.dupont`) 중 무엇이 움직였는지 말한다. 레버리지로 오른 ROE와 마진으로 오른 ROE는 다른 이야기다.
4. **`null` 지표는 숨기지 않는다.** `reason`을 그대로 옮긴다. 무차입 기업의 `interest_coverage: null`은 결함이 아니라 정보다.
5. **회계 정규화 결과는 항상 "분석 목적"이라고 붙인다.** `data.assumptions`가 그렇게 말한다. 정규화된 순이익을 보고 순이익처럼 쓰지 않는다.
6. **검산 결과를 본문에 한 줄 넣는다.** `balanced`, `reconciled`, `direct_indirect_difference`, `dupont.difference` 중 그 계산에 해당하는 것.
7. **신용 관련 문장에는 반드시 측도와 가정을 붙인다.** "1년 부도확률 0.12%(위험중립, 자산가치·변동성은 가정)".

## 카탈로그 호출 줄이기

`fintool_catalog`는 스펙 조회일 뿐 계산이 아니다. 계측에서 전체 도구 호출의 31%가 카탈로그였다.

- 이 문서에 **호출 예시가 있는 도구는 카탈로그를 부르지 않는다.** 예시를 그대로 쓰고 숫자만 바꾼다.
- `fsa`는 플래그가 100개가 넘어 카탈로그 응답이 크다. **모드별 필수 플래그가 위 표에 다 있으므로 부르지 않는다.**
- 예시가 없는 도구만 `fintool_catalog {"tool":"<이름>"}`으로 그 도구 하나를 받는다.
- 호출이 실패하면 오류 봉투가 무엇이 잘못됐는지 정확히 말해준다. 그것을 보고 고치면 되고 카탈로그를 따로 부를 필요가 없다.

## 함정

- **`fsa`의 필수 플래그는 값이 0이어도 명시해야 한다.** 무차입 기업의 `--interest-expense 0`을 생략하면 거절된다.
- **`--solve ratios`는 24개, `--solve cashflow`는 27개가 전부 필수다.** `ratios`는 `--metrics`로 그룹을 골라 그만큼만 요구하게 줄일 수 있다. 모드에 없는 플래그를 섞어도 거절된다.
- **`ratios`의 회계등식은 하드 거절, `cashflow`의 현금 연결은 경고다.** 전자는 `--tolerance`로 완화하고, 후자는 `reconciled: false`인 채 결과가 나오므로 직접 확인해야 한다.
- **`cashflow`는 손익 항등식을 강제한다.** 기타수익·지분법손익을 `--operating-expenses`에 순액으로 합치고 그 사실을 밝힌다.
- **`--adjustment inventory`는 한국 기업에 적용 불가다.** K-IFRS·일반기업회계기준 모두 LIFO를 금지한다.
- **일반기업회계기준은 `us-gaap`이 아니라 `ifrs`로 매핑한다.** 대신 `guidance`의 IAS 조문을 그대로 인용하지 말고 실제 적용 기준을 각주로 단다.
- **`credit`의 최상위 `default_probability`는 위험중립 값이다.** 실측도는 `--drift`를 줘야 `measures`에 생긴다.
- **`credit`의 자산가치·자산변동성은 DART에 없다.** 역산 기능이 없으므로 전부 가정이고, 그 사실을 매번 적는다.
- **신용등급을 추정하지 않는다.** fintool은 등급을 산출하지 않고, Merton PD는 등급별 실측 부도율보다 낮게 나온다.
- **VIF 위치가 모형마다 다르다.** OLS는 `data.diagnostics.vif`, 로지스틱은 `data.vif`(최상위)다.
- **로지스틱 분류 임계값은 0.5 고정이다.** 부도 표본이 드문 데이터에서 `accuracy`는 성능 지표가 아니다.
- **로지스틱 수렴 실패는 대개 완전분리다.** `--max-iterations`로 해결되지 않는다.
- **모든 `fsa` 비율은 `{value, formula}` 객체다.** 숫자는 `.value`에 있고 `null`이면 `reason`이 붙는다.
- 표본이 짧으면 회귀 계수·R²·부도예측 전부 신뢰구간이 넓다. `sample_size`를 함께 낸다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
