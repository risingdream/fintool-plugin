---
name: workforce-economics
description: fintool로 loaded 인건비, headcount·FTE, 휴가·공휴일·결근과 회의·교육 등 비생산시간을 반영한 available/productive capacity, 분모별 인건비율을 계산한다. 급여 외 복리후생·사용자가 확정한 4대 보험 등 employer on-cost·직접 고용비·credit의 중복 없는 rollup, 인력 계획의 시간 검산, cost per available/productive hour를 요청하면 사용한다. 제품·서비스·프로젝트 원가 귀속은 costing, 월별 손익·현금·런웨이는 startup-finance와 financial-model, 영업 quota·ramp·pipeline capacity는 sales-capacity, 확정 단위 원가의 공헌이익·가격 의사결정은 unit-economics와 pricing으로 보낸다.
---

# Workforce Economics

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

`workforce-economics`는 같은 기간·같은 인력 그룹에서 FTE, 시간 capacity, loaded employment cost를 각각 보존하고 서로 대조한다. 실행 가능한 입력은 카탈로그의 `input_contract.call_example`을 복사하고 실제 값만 바꾼다. workflow·batch 예시는 이 스킬의 `examples/`에 포함되어 있다.

## 먼저 라우팅한다

| 질문 | 도구 |
|---|---|
| 급여·benefit·employer on-cost·직접 고용비, FTE, leave·비생산시간 반영 capacity, 분모별 rate | `workforce-economics` |
| 제품·서비스·프로젝트에 소비된 시간과 객체별 원가·배부 | `costing` |
| 월별 매출·COGS·OPEX·현금·런웨이 | `startup-finance` → `financial-model` |
| quota·ramp·pipeline·required hires 등 영업 수용량 | `sales-capacity` |
| 확정된 단위 원가의 공헌이익·손익분기 | `unit-economics` |
| 가격·탄력성·수요·이익 변화 | `pricing` |
| 세금·원천징수·노무 적법성·법정 부담금·회계 인식 | 계산을 멈추고 관할 자료나 전문가에게 연결 |

월 인건비 총액만 이미 있고 재무계획에 넣으려는 질문에는 이 스킬을 강제하지 않는다. loaded cost 구성, 채용 timing/FTE, leave/capacity 또는 downstream rate의 분모가 필요할 때 사용한다.

## 입력 인터뷰

모르는 값을 0이나 업계 평균으로 채우지 않는다. 계산에 필요한 값이면 사용자에게 받고, 정말 모르는 값은 `unsupported` evidence로 남길 수 있는지 현재 계약을 확인한다.

1. **기간과 기준**: 각 period의 `[start,end)`와 `plan|actual` 중 하나를 확정한다. 한 실행에서 둘을 섞지 않는다.
2. **그룹과 인원**: role/team/cohort ID, label, `employee|contractor|mixed|other`, opening·closing headcount를 받는다. 이 값은 법적 고용형태 판정이 아니다.
3. **FTE와 gross capacity**: 같은 period의 full-time reference hours, regular scheduled hours, overtime hours를 각각 받는다. overtime은 gross capacity에는 들어가지만 FTE에는 들어가지 않는다.
4. **시간 차감**: 휴일·휴가·결근 같은 unavailable bucket과 회의·교육·관리 등 nonproductive bucket을 겹치지 않게 받는다. category의 의미는 사용자가 정하며 도구가 재분류하지 않는다.
5. **사용한 productive time**: plan이면 allocated productive hours, actual이면 actual productive hours를 받는다. paid hours는 선택 보고 분모이며 available/productive hours와 같다고 가정하지 않는다.
6. **원가 ledger**: cash compensation, employer on-cost, direct employment cost, allocated support, credit을 component로 나눈다. 합계에 더할 행은 `additive`, 이미 상위 금액에 포함된 설명 행은 `memo`다. credit은 음수 비용이 아니라 양수 credit component다.
7. **통화와 evidence**: 단일 ISO 4217 대문자 통화, 각 입력의 source/assumption/derived/unsupported 근거, provider·as-of·ref를 받는다.
8. **downstream scope**: `compensation_cost|employment_cost|fully_loaded_cost` 중 넘길 원가 범위와 available/productive-capacity 중 rate 분모를 명시한다.

paid leave가 급여에 이미 포함되어 있다면 금액 breakdown은 `memo`, 시간은 unavailable bucket이다. 같은 금액을 additive benefit으로 다시 더하지 않는다.

## 계산과 검산

- `fte = regular_scheduled_hours / full_time_reference_hours`
- `gross_capacity = regular_scheduled + overtime`
- `gross_capacity = unavailable + nonproductive + productive_capacity`
- `productive_capacity = used_productive + remaining_capacity`
- `employment_cost = compensation_cost + direct_employment_cost - credits`
- `fully_loaded_cost = employment_cost + allocated_support_cost`

그룹 합계의 rate와 ratio는 그룹별 비율의 단순 평균이 아니라 합계 분자÷합계 분모다. overbooking은 remaining capacity를 음수 그대로 보존하고 warning으로 설명한다.

## 결과를 인용하는 법

- 수치는 `fintool` 성공 봉투의 `data`만 인용하고 직접 재계산하거나 반올림한 값을 새 사실처럼 만들지 않는다.
- `calculation_hash`는 자르지 않고 전문을 보존한다.
- `evidence`, `warnings`, `reconciliation`, `explanations`를 함께 확인한다.
- 0분모 rate·ratio는 숫자 0이 아니라 `null`이다. 같은 경로의 explanation을 사용자에게 전달한다.
- stdout은 JSON 봉투 하나다. 사람이 읽는 경고는 stderr에도 나올 수 있으나 stdout에 섞이지 않는다.
- 입력 오류는 종료 코드 `2`, 실행 오류는 `1`, 성공은 `0`이다.

## downstream handoff

공개 `data.handoffs.*` packet에는 `source_tool`, `source_schema`, 전체 `source_calculation_hash`, `field_map`, `lossiness`, `evidence_refs`가 들어간다. 실제 downstream 입력을 만들 때는 source 봉투의 period, basis, currency, group과 사용자가 선택한 cost/rate scope도 함께 보존한다. 이 값들이 packet에 자동 추가됐다고 가정하지 않는다.

| 대상 | 넘길 값 | 손실·금지 사항 |
|---|---|---|
| `costing` | 선택한 loaded cost, available capacity, 계약상 지원되는 productive capacity/rate scope | available과 productive를 이름만 바꿔 넣지 않는다. support cost를 양쪽에서 다시 배부하지 않는다. billable recovery rate와 workforce rate를 더하지 않는다. |
| `financial-model` | 월별 total FTE, 선택한 cost scope 한 개, 사용자가 정한 `cogs|opex` | role 합계와 total을 동시에 넣어 이중계상하지 않는다. 월보다 짧은 period는 금액·시간을 월로 합산하고 FTE를 다시 계산한다. 회계 역할을 직무명으로 추론하지 않는다. |
| `sales-capacity` | period, eligible group의 FTE/headcount/active fraction 후보 | quota·ramp·pipeline·hiring lead time을 생성하지 않는다. P0 period-total 결과에서 roster를 복원하지 않는다. |

handoff는 원본 봉투를 무손실로 대체하지 않는다. 어떤 필드를 선택·집계·생략했는지 `field_map`과 `lossiness`에 적는다.

## 금지 가드레일

- BLS·Eurostat·채용 플랫폼 benchmark를 자동 입력하지 않는다. 사용자가 채택한 값만 source/as-of가 있는 assumption으로 넣는다.
- 40시간/주, 연 2,080·2,087시간, 12개월 균등 급여, 공휴일·휴가율·benefit율·employer tax·overtime·nonproductive 비율을 숨은 기본값으로 두지 않는다.
- utilization이나 plan/actual 차이의 원인을 휴가·성과·수요·ramp 탓으로 자동 추정하지 않는다.
- 실시간 FX나 평균환율을 선택하지 않는다. 외화 금액은 사용자가 환산 기준과 결과를 확정한 뒤 단일 모델 통화로 넣는다.
- employee/contractor 분류, overtime 대상, 법정 휴가, 최저임금, 원천징수, 사회보험, COGS/OPEX/자산/충당부채를 판단하지 않는다.

## P1로 남기는 범위

개인 roster·입퇴사일·주간 일정·calendar event 전개, plan-vs-actual 비교, support-pool allocation은 P0 결과에 있다고 주장하지 않는다. 필요한 경우 현재 period-total 계산과 별도 입력 요구사항까지만 정리한다.
