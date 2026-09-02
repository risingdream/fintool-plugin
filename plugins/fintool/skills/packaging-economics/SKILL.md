---
name: packaging-economics
description: 계정·좌석·사용량·패키지 block, 포함량, 할인, 직접 변동원가를 주어진 수량에 적용해 plan·portfolio 경제성과 기준/후보 산술 bridge를 fintool로 검산한다. 패키징 가격표 분석, 플랜 믹스, 무료층 비용, packaging comparison 요청에 사용한다. 가격탄력성 수요 추정은 pricing, LTV/CAC·단일 CVP는 unit-economics, 월별 손익·현금은 financial-model, 원가 산출·배부는 costing으로 보낸다.
---

# Packaging Economics

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

사용자가 관측했거나 명시적으로 가정한 account·seat·usage와 가격표를 한 기간에 대입한다.
수요, churn, migration 또는 패키지 변경의 인과효과를 추정하지 않는다. 상세 필드·출력·workflow
계약은 카탈로그 응답을 정본으로 삼고, 실행 가능한 fixture는 이 스킬의 `examples/`를 쓴다.

## 먼저 모드를 고른다

- 현재 또는 하나의 명시적 후보 스냅샷: `--spec`
- 같은 base spec의 숫자 가정 여러 개: `--spec` + `--batch`
- 사용자가 승인한 범위·분포에서 case 생성: `scenario` → batch
- 같은 plan universe의 두 스냅샷 차이: `--baseline-spec` + `--candidate-spec`
- 여러 도구의 typed 연결: workflow v2

비교 모드의 `volume → mix → monetization`은 입력된 두 스냅샷의 순서 의존 산술 분해다.
수요나 plan 이동의 원인을 설명하는 모형이 아니다.

## 입력은 단계적으로 확인한다

이미 받은 값을 다시 묻지 말고, 각 단계의 필수값이 빠졌을 때만 다음 질문으로 넘어간다.
누락값을 0, 시장 평균, 경쟁사 benchmark 또는 관행값으로 채우지 않는다.

### 1. 의사결정과 비교 범위

- 무엇을 검산하는가: 현재 packaging, 명시적 후보, batch sensitivity, baseline/candidate bridge
- 공통 통화와 `[start,end)` 분석 기간, `month|quarter|year|custom` grain은 무엇인가
- 값은 `observed_billing_period`, `forecast`, `contract_normalized` 중 어느 measurement basis인가
- 비교라면 두 spec의 plan ID 집합이 같고 각 plan account count가 양수인가

통화·기간·measurement basis·plan universe가 다른 비교를 자동 정규화하지 않는다.

### 2. cell과 수량

plan × segment × monetization status × 수량 profile이 같은 단위로 cell을 나눈다.

- cell stable ID, plan ID, segment ID, `free|paid|trial|comped`, account count
- 각 quantity의 stable ID, 값, 단위, `per_account|cell_total|billed_total`, 필요 시 aggregation
- 포함량 권리가 `per_account`인지 `pooled`인지

`cell_total`에 per-account entitlement를 적용하지 않는다. account row가 없으면 pooled 계산이나
billing 정본의 `billed_total`이 필요하다. `billed_total`은 이미 과금수량이므로 included quantity는
명시적 0이고 direct usage cost의 원수량으로 재사용하지 않는다.

### 3. charge component와 할인

- `flat_per_account`: unit price와 `CURRENCY/account[/period]`
- `per_unit`: quantity ID, explicit included quantity, entitlement scope, unit price
- `per_package`: 위 필드와 양수 block size
- 할인: `none|percentage|fixed_amount` 하나와 대상 component ID

tier, stacked discount, proration을 가장 가까운 P0 component로 근사하지 않는다.

### 4. 직접 변동원가와 fixed cost

structured cost는 account, active seat, raw usage, net revenue share 네 자리를 모두 명시한다.
이미 upstream에서 합산한 값이면 `direct_variable_cost_override`만 쓰고 structured cost와 섞지
않는다. shared fixed cost는 금액 또는 명시적 `null`이다. `null`은 0이 아니다.

원가율·배부 근거를 새로 계산해야 하면 먼저 `costing`으로 보낸다. allocated full cost를 direct
variable cost로 자동 간주하지 않는다.

### 5. evidence와 외부 대조

- 계산 입력을 뒷받침하는 `source|assumption|derived|unsupported` evidence
- 외부 ledger의 account, gross list charge, net modeled charge가 있으면 `declared_totals`
- 비교 후보의 account count·mix·usage가 관측값인지 승인된 가정인지

evidence가 없으면 `[]`을 명시할 수 있지만 `evidence_status=incomplete`를 결과에 보고한다.
현재 별도 evidence hash는 없으므로 calculation hash만으로 근거 동일성을 주장하지 않는다.

## 실행

`fintool_catalog {"tool":"packaging-economics"}`의 `input_contract.call_example`과 typed ports를
정본으로 삼는다. 단일 실행은 catalog 예시의 `flags.spec` 객체를 실제 값으로 바꿔
`fintool_run {"tool":"packaging-economics","flags":{"spec":{"version":"packaging-economics/v1"}}}`
형태로 호출한다. 기준·후보 bridge는 `flags.baseline-spec`과 `flags.candidate-spec`, 명시적 batch는
`flags.spec`과 `flags.batch`에 각 JSON 객체를 넣는다.

scenario와 workflow는 다음 fixture를 그대로 실행할 수 있다.

- [`examples/scenario.json`](examples/scenario.json): 승인된 두 숫자 범위의 OAT case
- [`examples/workflow.json`](examples/workflow.json): scenario cases를 typed batch port로 연결
- [`examples/invalid.json`](examples/invalid.json): 통화 단위 오류와 종료 코드 2 확인

batch param은 base spec의 기존 numeric leaf만 stable-ID 점 경로로 덮는다. 문자열 enum, 단위,
배열 구조, 새 plan·component를 batch overlay로 바꾸지 않는다. 구조가 다른 후보는 별도 strict spec으로
계산하되 P0 bridge는 같은 plan universe만 비교한다.

## 결과를 읽는 순서

1. 봉투 `ok`, tool, error code와 종료 코드를 확인한다.
2. `normalized_inputs`에서 실제 계산된 통화·기간·cell·quantity·component·cost를 확인한다.
3. cell의 component raw/included/billable quantity, gross, eligible, discount, net, direct cost,
   contribution을 확인한다.
4. plan과 portfolio의 account·revenue·contribution mix, all/paid ARPA, operating income,
   fixed-mix break-even을 확인한다.
5. `reconciliation.balanced`와 외부 대조의 difference를 확인한다.
6. evidence status, warnings, explanations, `calculation_hash`를 함께 보고한다.

`null`을 0으로 읽지 않는다. eligible charge 0, net charge 0, account denominator 0,
`shared_fixed_cost=null`, 비양수 weighted contribution은 대응 ratio·ARPA·operating income·
break-even을 `null`로 만들 수 있고 `explanations`가 사유를 남긴다.

## bridge 해석

revenue와 contribution 각각 다음을 함께 제시한다.

- baseline, candidate, delta
- 고정된 `order=[volume,mix,monetization]`
- volume, mix, monetization, difference, balanced
- baseline·candidate scenario ID와 bridge `calculation_hash`

해석 문장은 “입력된 후보 account 수의 산술 volume 귀속”, “입력된 후보 plan share의 산술 mix
귀속”, “plan별 account당 modeled amount 차이의 monetization 귀속”으로 쓴다. “가격 때문에
고객이 늘었다”, “패키지가 migration을 유발했다” 같은 인과 문장을 쓰지 않는다.

monetization에는 순수 단가뿐 아니라 사용량, entitlement, discount 변화가 함께 들어갈 수 있다.
contribution bridge의 monetization에는 직접원가 변화도 들어갈 수 있지만 revenue bridge에는
들어가지 않는다. 순수 가격효과가 필요하면 다른 값을 고정한 명시적 candidate spec을 추가하고도
그것을 반사실적 인과효과라고 부르지 않는다.

## 결과 전달

최소한 다음을 보고한다.

- 분석 목적, 통화, 기간, measurement basis, 관측값/가정 구분
- cell·plan·portfolio의 net charge, direct variable cost, contribution과 주요 mix
- shared fixed cost, operating income, fixed-mix break-even의 scope와 조건
- 할인 대상과 effective discount, quantity basis와 entitlement scope
- reconciliation 결과, evidence status, warnings·null explanations, calculation hash
- 비교라면 고정 order의 bridge와 비인과성 경계
- downstream handoff에서 사라지는 세부 정보와 새로 필요한 입력

## Routing과 lossiness

| 도구 | 보낼 때 | 사라지거나 새로 필요한 것 |
| --- | --- | --- |
| `pricing` | 가격탄력성으로 단일 가격의 수요 반응을 추정 | plan/component/entitlement/discount/mix가 단일 price·units·variable cost로 축약된다. 비교 unit과 cohort 승인이 필요 |
| `unit-economics` | subscription LTV/CAC 또는 단일 transaction CVP | ARPA를 넘길 수 있지만 churn·CAC가 새로 필요하다. contribution margin은 gross margin으로 자동 매핑할 수 없고 portfolio 집계 시 cell·plan trace 손실 |
| `financial-model` | 선택 결과를 월별 손익·현금으로 전개 | snapshot을 월별 driver로 바꾸는 성장·churn·cash timing·COGS/OPEX·회계 가정이 새로 필요 |
| `costing` | 계정·좌석·사용량 원가율과 배부 trace를 산출 | packaging에 rate/override만 넘기면 BOM·capacity·pool trace가 손실되므로 upstream hash와 evidence 유지 |

handoff는 자동 변환 허가가 아니다. 사용자가 단위, 기간, scope, 회계 역할을 확인하고 downstream
도구의 입력·evidence 계약을 새로 충족해야 한다.

## 금지 guardrail

- 누락값을 0, 시장 평균, 경쟁사 benchmark로 채우기
- 자동 FX·기간 환산, 수요·churn·migration 예측
- 목표 이익이나 benchmark에서 패키지를 자동 역설계하기
- bridge를 causal effect, elasticity 또는 실험 결과로 설명하기
- tier·stacked discount·proration·신규/폐지 plan bridge를 P0로 근사하기
- structured direct cost와 override를 함께 넣기
- billing modeled charge를 invoice, 세금, 법정 revenue로 단정하기
- hash가 같다는 이유로 evidence도 같거나 입력이 사실이라고 주장하기
- 추천점수 하나로 가격·원가·품질·시장·법률 판단을 합치기

P0 밖이면 계산 가능한 것처럼 축약하지 말고 표현되지 않는 의미를 밝힌 뒤 적절한 도구 또는
billing/ERP 정본으로 handoff한다.
