---
name: sales-capacity
description: fintool로 영업 목표·quota·credited actual·ramp·attainment·pipeline을 대조해 예상 period-end capacity, gap, 필요 pipeline과 profile별 required hires를 계산한다. 영업 인원 몇 명이 필요한지, quota 달성률, ramp 반영 capacity, pipeline-to-gap을 묻는 경우 사용한다. lead funnel·CAC·ROAS는 gtm-economics, 인건비·roster·채용비는 workforce-economics, revenue·cash timing은 financial-model, LTV/CAC·공헌이익은 unit-economics로 보낸다.
---

# Sales Capacity

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

`sales-capacity`는 하나의 measure·unit·`[start,end)` 기간에서 target, assigned quota, net credited
actual, remaining capacity, pipeline snapshot과 hiring profile을 분리해 계산한다.
실행 가능한 입력은 카탈로그의 `input_contract.call_example`을 복사해 만들고, 전체 필드·공식·출력
계약은 같은 카탈로그 응답을 따른다.

## 먼저 라우팅한다

| 질문 | 도구 |
|---|---|
| quota·ramp·attainment·credited actual, 예상 period-end, capacity gap, required reps | `sales-capacity` |
| lead/MQL/SQL/opportunity funnel, CAC·ROAS, top-of-funnel 역산 | `gtm-economics` |
| 급여·benefit·채용비·roster·근무시간·productive hours | `workforce-economics` |
| bookings/new ARR가 revenue·cash로 전환되는 timing | `financial-model` |
| CAC·LTV/CAC·payback·공헌이익 | `unit-economics` |

질문이 “영업 인원 몇 명 필요해”여도 quota, active fraction, ramp, expected attainment가 없으면
benchmark로 채우지 않는다. 사용자가 승인한 값을 받아 profile별 산술 시나리오를 낸다.

## 입력 인터뷰

1. **measure와 기간**: `bookings|new_arr|revenue|deal_count|custom`, `money|count`, 단일 unit,
   `[start,end)`, cutoff `as_of`, timezone, grain을 확정한다.
2. **target과 quota**: target definition, stable-ID quota line, included `sum`과 memo 성격의
   `exclude`, direct quota 또는 `headcount × quota_per_rep` 중 하나를 받는다.
3. **actual**: cutoff까지 gross credited, reversal, crediting rule을 같은 unit으로 받는다.
4. **current capacity**: stable-ID line마다 headcount, `full_period_quota|effective_quota`, quota per
   rep, active fraction basis, ramp factor, expected attainment를 받는다.
5. **pipeline**: snapshot, 분석기간과 같은 close window, 포함 category, aggregate gross/weighted
   또는 stable-ID opportunity를 받는다. probability·stage conversion·win rate·target ratio는 source
   값을 명시한 경우만 사용한다.
6. **hiring profile**: role·segment·cohort별 quota, active fraction, ramp, expected attainment를 받는다.
   여러 profile을 자동 혼합하거나 최적화하지 않는다.
7. **evidence**: 모든 material numeric field를 stable path로 `source|assumption|derived|unsupported`
   evidence에 연결한다. source는 provider와 as-of, assumption은 note와 as-of가 필요하다.

## 계산과 해석

- `assigned_quota = sum(included quota lines)`
- `net_credited = gross_credited - reversals`
- `quota_attainment = net_credited / assigned_quota`
- `full_period productive_fte = headcount × active_fraction × ramp_factor`
- `full_period expected_capacity = headcount × quota_per_rep × active_fraction × ramp_factor × expected_attainment`
- `effective expected_capacity = headcount × effective_quota_per_rep × expected_attainment`
- `expected_period_end = net_credited + sum(line expected_capacity)`
- `capacity_gap = max(0, target - expected_period_end)`
- `pipeline_to_gap = eligible_open_pipeline / remaining_target`
- `gap_to_pipeline = remaining_target / eligible_open_pipeline`
- `required_pipeline_at_win_rate = remaining_target / supplied_win_rate`
- `required_productive_capacity = capacity_gap`
- `capacity_per_new_hire = quota_per_rep × active_fraction × ramp_factor × expected_attainment`
- `required_hires_continuous = capacity_gap / capacity_per_new_hire`
- `required_hires_integer = ceil(required_hires_continuous)`

team attainment와 coverage는 line ratio의 평균이 아니라 합산 분자÷합산 분모다. `effective_quota`는
active와 ramp를 source quota에 이미 반영한 basis이므로 두 값이 모두 `1`이어야 한다. 다시 곱하면
입력 오류다.

## 결과를 인용하는 법

- 성공 봉투의 `data`만 인용하고 `calculation_hash`는 전문을 보존한다.
- `evidence_status`, `fully_explained`, `warnings`, `explanations`, `reconciliation`을 수치와 함께 본다.
- 정상 reconciliation의 가용 check는 `difference:0`이어야 한다. weighted probability가 완전하지
  않으면 해당 check는 `not_available`일 수 있다.
- 0분모 ratio는 숫자 `0`으로 바꾸지 말고 `null`과 같은 경로의 explanation을 전달한다.
- `pipeline_to_gap`과 `gap_to_pipeline`은 방향이 다르므로 “coverage” 한 단어로 합치지 않는다.

## Typed handoff

`data.handoffs.*`는 자동 변환된 downstream spec이 아니라 source path와 손실을 밝히는 후보
packet이다. 각 packet의 `source_tool`, `source_schema`, 전체 `source_calculation_hash`, JSON Pointer
`field_map`, `lossiness`, `evidence_refs`를 보존한다.

| packet | 의미 | 금지 사항 |
|---|---|---|
| `workforce_input` | workforce FTE·productive capacity가 headcount·active-fraction 후보가 될 수 있음 | cost·FTE schedule을 quota·ramp·attainment로 승격하지 않음 |
| `gtm_economics_input` | funnel stage가 opportunity 후보가 될 수 있음 | CAC·ROAS·funnel을 quota credit·win rate로 승격하지 않음 |
| `workforce_output` | capacity gap과 continuous/integer required hires 후보 | lead time·start date·roster·cost·attrition을 생성하지 않음 |

`financial-model`에는 계약 시작·term·billing·revenue recognition·collection schedule이 별도로
필요하다. `unit-economics`에는 CAC kind/scope, gross margin, retention·LTV basis가 별도로 필요하다.
sales-capacity 결과만으로 이 필드를 만들지 않는다.

## 금지 가드레일

- benchmark, 3×·4× pipeline rule, 평균 quota, ramp, attainment, win rate를 자동 입력하지 않는다.
- 결과 차이의 원인을 생산성·영업력·시장 수요·채용 부족으로 자동 추정하지 않는다.
- profile별 required hires를 승인된 headcount 권고나 자동 role mix로 표현하지 않는다.
- 여러 통화·measure·unit을 합치거나 실시간 FX를 선택하지 않는다.
- stable ID를 배열 index로 대체하거나 source path·lossiness를 버리지 않는다.
- actual funnel win을 quota credit으로, attributed revenue를 bookings/new ARR로 간주하지 않는다.

## P1과 보류 범위

P0는 단일 period-total line과 hiring profile 산술이다. 입사일별 hiring cohort, 다기간 ramp curve,
attrition/backfill, role mix 최적화, 월별 roster·lead time, compensation·recruiting cost, 다통화 FX는
구현됐다고 주장하지 않는다. 이 값이 필요하면 workforce·GTM·financial model의 source schedule을
별도로 받고, P1 계약이 생기기 전에는 현재 P0 숫자에서 복원하지 않는다.
