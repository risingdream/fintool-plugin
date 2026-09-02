---
name: costing
description: fintool MCP로 제품·서비스·프로젝트·고객의 원가를 구성요소부터 쌓는다. BOM 원가, 제품 원가, 서비스 제공원가, 건당 원가, 프로젝트 수익성, 고객별 원가, 채널 원가, cost-to-serve, 간접비 배부, 배부 기준, 원가 driver, 시간당 원가율, capacity·가동률, 외주·API·클라우드 원가, 표준원가 차이(재료 가격·수량, 노무 임률·능률, 간접비 예산·조업도), 배합·수율 차이, 2단계 ABC(활동기준원가), 복수 제품 믹스 손익분기를 요청하면 이 스킬을 쓴다. 실제·계획 GTM 지출과 퍼널의 채널 CAC·ROAS·ROMI는 gtm-economics, 계획 대비 실적의 매출 PVM·영업이익 브리지는 operating-variance로 보낸다. 이미 단위 변동비·고정비가 있고 손익분기만 필요하면 unit-economics, 가격탄력성 손익은 pricing, 월별 매출·COGS·현금·런웨이는 startup-finance(financial-model), 연간 3개년 BP는 startup-finance(business-plan)로 보낸다. 재고평가·법정 원가명세서·세무 신고는 범위 밖이다. 숫자는 추정하지 말고 fintool 봉투만 인용한다.
---

# Costing

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

원격 MCP: `fintool_catalog` → `fintool_run`. 로컬 바이너리나 계산기로 원가를 만들지 않는다.

이 스킬 범위: 원가 구성요소를 쌓아 **원가를 만드는** 일. `costing` 도구 하나를 쓴다.
범위 밖은 아래 라우팅 표에 있다.

## 라우팅 — 무엇을 이미 알고 있는가로 가른다

| 사용자가 이미 가진 것 | 알고 싶은 것 | 어디로 |
|---|---|---|
| 재료·시간·사용량·간접비 같은 **구성요소** | 단위원가·건당 원가·고객별 원가 | **`costing`** (이 스킬) |
| 매체비·campaign·sales/marketing 지출과 cohort 퍼널 | 채널 CAC·ROAS·ROMI·목표 리드/예산 | `gtm-economics` |
| 이미 합산된 단위 변동비·고정비 | 손익분기·LTV/CAC·공헌이익 | `unit-economics` |
| 기준 가격·판매량·탄력성 | 가격 변경의 손익 | `pricing` |
| 월별 매출·원가율·채용·조달 계획 | 월별 손익·현금·런웨이 | `startup-finance` → `financial-model` |
| 연간 성장률·원가율 | 3개년 BP와 시나리오 | `startup-finance` → `business-plan` |
| 표준원가표와 실제 투입 | 가격·수량·임률·능률·조업도 차이 | **`costing`** (`standards` 블록) |
| 계획 P&L과 마감 실적 | 매출 PVM·영업이익·현금 브리지 | `operating-variance` |
| 원가 결과 | 재고평가·법정 원가명세서·세무 신고 | **범위 밖.** 전문 검토를 안내한다 |

`costing`의 출력은 `unit-economics`·`financial-model`의 **입력**이다. 원가를 만들었으면
그다음 판단은 그쪽으로 넘긴다. 반대로 사용자가 이미 "단위 변동비 39,480원"을 들고 있으면
이 스킬을 거치지 않는다.

## 첫 호출 계약

`fintool_catalog {"tool":"costing"}`가 공통 뼈대 계약과 **모델별 블록 계약**을 함께 낸다.
`input_contract.call_example`을 복사해 값만 바꾸고, 고른 모델의 스키마는
`input_contract.models[]`에서 본다. 네 모델의 예시가 전부 그대로 실행된다.

evidence의 입력 경로는 `fields` 배열에 `spec.<path>`로 적고 `kind`는
`source|assumption|derived|unsupported` 중 하나를 쓴다. `kind:"derived"` 항목에는
파생 근거인 `ref`가 필수이며, 설명문인 `note`를 `ref` 대신 쓰지 않는다.

## 모델 고르기

| 사용자의 말 | model | 받는 블록 |
|---|---|---|
| "이 제품 하나 만드는 데 얼마" | `product` | `product` (BOM·공정·폐기) |
| "이 서비스 한 건 처리 원가" | `service` | `service` (역할 시간·사용량·외주) |
| "이 프로젝트 남았나" | `project` | `project` (+ 경비·청구) |
| "어느 고객·채널이 비싼가" | `cost_to_serve` | `cost_to_serve` (직접비 + 배부) |

model과 다른 블록을 함께 주면 거절된다.

## 어느 모델에도 붙는 확장 블록

| 사용자의 말 | 블록 | 무엇을 받나 |
|---|---|---|
| "표준 대비 얼마나 벌어졌나" | `standards` | 표준값만. 실제는 이미 스펙에 있는 줄을 참조한다 |
| "이 원가가 어느 활동을 거쳐 왔나" | `abc` | resource pool → activity → cost object 2단계 |
| "제품 두 개를 같이 팔면 손익분기가 몇 개" | `sales_mix` | 배합 비율. 고정비는 엔진이 계산한 값을 쓴다 |

- **`standards`는 실제를 다시 받지 않는다.** `component_id`·`operation_id`·`role_id`·
  `pool_id`로 실제 줄을 가리킨다. 실제 수량·단가를 표준 블록에 또 적으면 두 값이 어긋났을 때
  어느 쪽이 참인지 알 수 없어 거절 대상이 아니라 조용한 오류가 된다.
- **차이의 부호는 양수 = 불리, 음수 = 유리다.** 결과의 `sign_convention`을 함께 인용한다.
  차이의 **원인**은 이 도구가 말하지 않는다. 산술 분해일 뿐이라고 명시한다.
- **재료 수량 차이는 usage와 mix+yield 중 하나만 고른다.** `mix + yield`는 `usage`와 같은
  값이라 함께 더하면 이중계상이다. 고른 분해만 채워지고 나머지는 `null`이므로,
  `null`이 아닌 필드만 더해 총차이를 다시 만든다.
- **`abc`의 activity는 금액을 받지 않는다.** 활동 원가는 1단계에서 흘러들어온다.
  activity가 activity를 소비하는 경로가 없어 순환 배부는 요청받아도 만들 수 없다 —
  상호 서비스부문 배부는 범위 밖이라고 말한다.
- **`sales_mix`의 고정비를 사용자에게 묻지 않는다.** 이 실행이 계산한 배합 객체들의
  고정비 합을 쓴다. 배합 밖 고정비가 남으면 경고가 나오므로 그 경고를 그대로 전한다.
- **일정한 배합은 가정이다.** `fields`에 `"sales_mix"`를 담은 evidence가 없으면 거절된다.
  근거를 지어내지 말고 사용자에게 어느 기간의 실제 배합인지 묻는다.

## 인터뷰 순서

숫자를 받기 전에 아래 순서로 확인한다. 사용자가 이미 말한 것은 다시 묻지 않는다.

1. **결정**: 원가를 알아서 무슨 결정을 하려는가 — 가격, 외주 여부, 제품 중단,
   고객·채널 정리, 프로젝트 견적 중 무엇인가. 이 답이 model과 원가 객체를 정한다.
2. **원가 객체와 기간**: 제품 1개인가 배치인가, 서비스 1건인가 월 전체인가.
3. **완료 단위**: 생산·처리·납품된 **양품** 수량. 착수량과 완료량을 구분한다.
   착수량과 수율을 주면 양품은 엔진이 계산한다.
4. **직접 추적비**: 재료 수량×단가, 역할별 시간×원가율, 외주, API·클라우드 사용량×단가.
5. **capacity와 수율**: 제품은 yield·scrap·recovery, 서비스는 available·tracked·billable
   hours. **필요할 때만** 묻는다.
6. **공유비용**: 직접 추적할 수 없는 비용만 pool로 모은다. 그 비용을 **무엇이 유발하는지**
   driver와 분모를 함께 묻는다.
7. **수익성**: 가격·매출이 있으면 공헌이익(변동비만 차감)과 full-cost 이익(배부 고정비까지
   차감)을 나눠 보여준다.
8. **검산과 인계**: `reconciliation`, 미배부액, 근거 없는 입력을 먼저 보여주고
   `unit-economics`·`financial-model`로 넘길지 확인한다.

## 가드레일

- **업종 평균을 만들지 않는다.** 수율·가동률·배부율의 "보통 값"을 지어내지 않는다.
  모르면 모른다고 하고 사용자에게 묻는다.
- **배부 기준을 승인 없이 정하지 않는다.** "보통 매출 비중으로 배부한다" 같은 제안을
  입력에 넣기 전에 사용자 확인을 받는다. pool driver에는 evidence가 필수이고
  (없으면 `missing_evidence`로 거절된다) 그 근거는 사용자가 준 것이어야 한다.
  근거를 만들어 채워 넣어 오류를 통과시키지 않는다 — 그 순간 배부 기준의 출처가 사라진다.
- **overhead를 두 번 더하지 않는다.** 사용자가 준 시간당 원가율에 간접비가 이미 포함돼
  있는지 확인한다. 포함돼 있는데 같은 pool을 또 배부하면 원가가 부풀려진다.
- **폐기율을 두 번 반영하지 않는다.** BOM 수량이 이미 폐기를 반영한 투입량(`gross`)인지
  완제품에 남는 양(`net`)인지 물어 `quantity_basis`로 구분한다.
- **capacity 원가를 투입원가로 착각하지 않는다.** resource의 `loaded_cost`는 rate의
  분자다. `reconciliation.input_cost`에 들어가지 않으며 미회수분은 `capacity` 섹션의
  분석값이다. 역할은 `cost_rate`와 `resource_id` 중 하나만 갖는다.
- **경영관리용임을 표시한다.** 결과의 `purpose`는 `managerial`이다. 재고평가나
  외부 재무보고에 그대로 쓸 수 있다고 말하지 않는다.
- **정의되지 않은 값을 0으로 채우지 않는다.** 근거 없는 입력은 evidence를
  `unsupported`로 남긴다.

## 결과에서 반드시 먼저 보여줄 것

```
1. reconciliation — assigned + unassigned = input, difference는 0
2. unassigned_cost가 0이 아니면 그 이유(미사용 capacity인지 미배부인지)
3. allocations[].driver_rate와 driver_usage — 배부가 어떻게 나뉘었는지
4. capacity[].unrecovered_cost — 회수되지 않은 인력 원가(있으면)
5. evidence가 unsupported인 항목

standards를 썼으면 variances.sign_convention과 totals를 먼저 보여주고,
abc를 썼으면 activities[].activity_cost와 driver_rate로 원가가 지나온 경로를 보여준다.
sales_mix를 썼으면 break_even_bundles와 함께 fixed_cost가 무엇의 합인지 밝힌다.
```

`difference`는 언제나 0이다. 0이 아니면 그것은 도구의 버그이므로 결과를 인용하지 말고
그 사실을 알린다.

## 단위원가 정밀도

금액만 소수 둘째 자리에서 반올림하고 단위원가·배부율·비율은 원정밀도로 나온다.
`unit_full`이 `18.333333333333332`로 오면 그대로 인용하고, 사람이 읽을 자리에서만
반올림하되 반올림했다는 사실을 적는다. 반올림한 단위원가를 다시 곱해 총원가를 만들지 않는다.

## 민감도 — `--batch`

원가 driver를 흔들어 단위원가 분포를 본다. **`params`의 키가 플래그 이름이 아니라
스펙 안의 점 경로다.** 배열은 원소 `id`로 내려간다.

```json
{"tool":"costing","flags":{"spec":{"...":"..."},"batch":{"cases":[
  {"label":"base","params":{}},
  {"label":"hours_up","params":{"service.roles.role.reviewer.hours":100}},
  {"label":"pool_up","params":{"cost_pools.pool.support.amount":3000000}}]}}}
```

집계는 `aggregate.metrics`의 `by_object.<객체 id>.unit_full`에서 읽는다. 배열 안에 있는
단위원가는 교차 케이스 집계가 건너뛰므로 `by_object` 색인이 그 자리를 대신한다.

`scenario`로 케이스를 만들었으면 그 `run_id`를 `flags.batch`에
`{"$ref":"run_..."}`로 넘긴다. 케이스 목록을 눈으로 읽어 다시 타이핑하지 않는다.

## 함정

- **원가를 채팅에서 곱하지 않는다.** 200kg × 5원을 직접 계산해 `amount`로 넣으면
  수량과 단가가 결과에서 사라져 무엇을 바꿔야 원가가 내려가는지 알 수 없게 된다.
- **단가는 단위째 준다.** `unit_cost.unit`의 분모와 `quantity`의 `unit`이 다르면
  계산 전에 거절된다(`KRW/api_call` 단가에 token 수량을 곱할 수 없다).
- **매출은 한 곳에서만 정한다.** `cost_objects[].price`·`revenues`·`project.billing`
  중 둘이 겹치면 거절된다.
- **`actual_total_usage`는 사용량 합이 분모와 같아야 한다.** 다르면
  `allocation_mismatch`다. 사전 배부율을 쓰려면 `normal_capacity`·`budget_capacity`로
  바꾸고, 그때 미배부액은 단위원가에 다시 얹지 않는다.
- 원격 MCP에는 파일시스템이 없어 `@파일.json`을 쓸 수 없다. 같은 스펙을 여러 번 쓰면
  `fintool_save`로 저장하고 `flags.spec`에 `{"$model":"이름"}`으로 가리킨다.
- 해시는 자르지 않고 전문으로 인용한다.

## 아직 없는 것

다단계 BOM roll-up, 변동 간접비의 spending·efficiency 분해, 순환 서비스부문 상호배부와
3단계 이상의 원가 네트워크는 구현되지 않았다(`abc`는 2단계까지다). 사용자가 요청하면
없다고 말하고 지금 범위로 답할 수 있는 부분만 계산한다.

계획 대비 실적의 매출 PVM·영업이익 브리지·현금 브리지는 이 도구가 아니라
`operating-variance`가 맡는다. `standards`는 **표준원가표 대비 실제 투입**을 보는 것이지
예산 P&L 대비 마감 실적을 보는 것이 아니다.
