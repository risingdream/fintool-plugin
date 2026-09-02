---
name: vendor-tco
description: 벤더 견적 비교, 공급사 계약 TCO, 최소 spend·usage 약정, 미사용 약정, 고정료·구축·전환·exit 비용, 사용량 손익분기를 fintool로 계산한다. 시장·경쟁사 가격 조사 자체는 startup-competitors, 확정 비용의 원가 귀속은 costing, NPV·회수기간은 capital-budget, 회사 재무 전개는 financial-model로 보낸다.
---

# Vendor TCO

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

구매자 관점에서 같은 통화·같은 사용량 단위·같은 완전 기간의 공급사 계약을 비교한다.
원격 MCP의 `vendor-tco`를 사용한다. 처음에는
`fintool_catalog {"tool":"vendor-tco"}`로 `input_contract.call_example`과 workflow port를
확인하고, 예시의 `spec`을 실제 값으로 교체해 `fintool_run`으로 실행한다. 숫자는 성공 봉투만
인용하고, 기능·품질·위험이 동등하다는 판단과 입력 근거는 계산 밖에 분리해 둔다.
상세 입력·출력 계약과 실행 예제는 카탈로그의 `input_contract.call_example`과 workflow port를
정본으로 삼는다.

## 첫 확인

첫 계산 전에 다음을 한 번에 확인한다. 빠진 값을 0이나 관행값으로 채우지 않는다.

- 비교안의 기능·서비스 범위·SLA가 의사결정 목적상 동등한가
- 공통 통화와 사용량 단위는 무엇인가
- 분석 기간과 각 fulfillment period는 어디서 어디까지인가. 모두 완전 기간인가
- 기간별 실제 또는 명시적 시나리오 사용량은 얼마인가
- 각 option의 기간 고정료, 약정 유형(`none|spend|usage`), 약정액·약정수량, 약정 단가, 초과 단가는 얼마인가
- 약정 fulfillment가 `period`이고 rollover가 `false`인가. true-up·이월이 있다면 P0로 계산하지 않는다
- setup·switch-in·switch-out 비용의 포함 범위와 timing은 무엇인가
- 견적·계약·사용량·one-time 비용을 뒷받침하는 source 또는 승인된 assumption evidence는 무엇인가

## 계산 흐름

1. `fintool_catalog {"tool":"vendor-tco"}`에서 현재 `input_contract.call_example`, 플래그,
   workflow port를 확인한다. 문서 예제보다 설치된 catalog가 우선한다.
2. 입력을 명시적 `vendor-tco/v1` JSON으로 만든다. 한 통화, 한 flat meter, 연속된 완전
   period, 전 기간을 덮는 계약만 허용한다.
3. 실제 사용량 경로의 option별 period breakdown과 TCO를 계산한다.
4. 요청한 pair에 대해서만 `constant_usage_break_even`을 계산한다. 실제 사용량 TCO 결과와
   일정 사용량 손익분기 가정을 섞지 않는다.
5. 봉투의 evidence coverage, warnings·explanations, reconciliation, `inputs_hash`·`result_hash`·
   `calculation_hash`·`evidence_hash`를 확인한 뒤 결과를 설명한다.

한 계약 비교는 단일 `spec` 호출을 쓴다. 여러 명시적 숫자 case는 base spec의 numeric leaf를
stable-ID 점 경로로 덮는 공용 `--batch`를 쓰고, 사용자가 범위·분포를 승인한 경우에만
`scenario` case를 연결한다. 여러 도구를 연결해야 할 때는 workflow v2의 typed `spec` JSON
port와 `scenario_cases` batch port를 사용한다. 사용법은 도구 문서의 검증된 fixture를 복사한다.

## 읽는 법

- `covered_quantity`: 실제 사용량 중 약정으로 덮인 수량
- `unused_committed_quantity`: 그 period에 쓰지 못하고 소멸한 약정수량
- `overage_quantity`: 약정 초과 수량. `none`에서는 전체 실제 사용량이 이 계산 경로를 쓴다
- `billable_usage_cost`: 사용한 약정원가 + 미사용 약정원가 + 초과원가
- `recurring_total`: 모든 period의 고정료와 billable usage cost 합계
- `one_time_total`: setup + switch-in + switch-out
- `tco`: recurring + one-time인 비할인 명목 합계
- `effective_tco_per_usage`: 총 사용량이 0이면 숫자가 아니라 `null`과 explanation

손익분기는 `points`, `intervals`, `all`, `none`을 그대로 보존한다. 여러 교점이나 동률 구간을
임의의 숫자 하나로 축약하지 않는다. Google 25% 할인 특수 사례의 75%는 보편 benchmark가
아니라 `q* = Q × committed_rate / on_demand_rate`의 특정 입력 결과다.

## 결과 전달

최소한 다음을 함께 제시한다.

- 비교 범위: 통화, usage unit, period 수, 실제 사용량인지 시나리오인지
- option별 recurring, setup, switch-in, switch-out, one-time, TCO와 실제 TCO 차이
- 약정별 covered·unused·overage 수량과 비용. 미사용 약정액을 절감으로 표현하지 않는다
- constant-usage break-even의 가정과 모든 point·interval
- evidence coverage와 missing·unsupported·conflict·stale warning
- `calculation_hash`와 `evidence_hash`. 숫자가 같아도 근거가 바뀌면 evidence hash가 달라질 수 있다
- downstream handoff에 넘길 명시적 비용 일정과 아직 결정하지 않은 회계·할인 가정

## Routing과 handoff

- 공급사·경쟁사 탐색, 공개 가격·기능·SLA 리서치: `startup-competitors`
- 확정된 vendor 비용의 제품·서비스·고객 원가 귀속: `costing`
- 절감 편익을 포함한 NPV·IRR·회수기간: `capital-budget`
- 선택 계약의 COGS·OPEX·CAPEX 분류와 회사 손익·현금 전개: `financial-model`

이 handoff는 숫자를 자동 변환한다는 뜻이 아니다. 사용자가 회계 역할, 현금시점, 할인율,
편익, 배부 driver를 명시하고 각 downstream 도구의 evidence 계약을 새로 충족해야 한다.

## 금지

- 공급사 가격·시장 평균의 자동 benchmark 또는 누락값 기본 입력
- 과거 사용량에서 미래 사용량을 자동 예측하거나 최적 약정·vendor를 자동 추천
- 자동 FX, 자동 갱신, proration, rollover, true-up, credit, tier·multi-SKU 등 P1 의미론
- 계약 법률 해석, 세금·VAT 판단, 법정 회계·자본화 판단
- NPV·IRR·회수기간을 이 도구 결과라고 계산하거나 설명
- 기능·품질·보안·가용성·위험을 점수화해 TCO와 합친 단일 추천점수

입력이 P0 밖이면 근사하지 말고 어느 필드가 왜 표현되지 않는지 밝힌 뒤 적절한 handoff 또는
명시적 별도 시나리오를 제안한다.
