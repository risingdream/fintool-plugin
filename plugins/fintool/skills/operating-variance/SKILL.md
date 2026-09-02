---
name: operating-variance
description: fintool MCP로 동결된 계획·예산과 마감 실적의 차이를 산술적으로 설명한다. 계획 대비 실적, 예산 대비 실적, PVM, 가격·수량·믹스, 영업이익 브리지, 현금 브리지 요청에 사용한다. 계획·forecast 작성은 startup-finance, 미래 가격 의사결정은 pricing, 상세 원가 구축은 costing, 실제 CFO 계산·검산은 fsa로 보낸다.
---

# Operating Variance

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

원격 MCP의 `operating-variance`를 사용한다. 처음에는
`fintool_catalog {"tool":"operating-variance"}`로 `input_contract.call_example`을 확인하고,
그 예시를 복사해 값만 바꾼 뒤 `fintool_run`으로 실행한다. 수치는 성공 JSON 봉투에서만
인용하고, 필드·공식·workflow port는 카탈로그의 최신 계약을 따른다.

## 경계와 라우팅

이 스킬은 같은 기간·범위·통화·scale로 고정한 plan과 actual snapshot을 받아 다음 세 가지를
forward reconciliation한다.

- 매출: `actual_quantity_price_first` 가격·수량·믹스(PVM), 명시적 scope, 수량 없는 조정
- 영업이익: amount-only income·expense·contra-revenue의 `actual - plan` 브리지
- 현금: upstream이 분류하고 부호를 정한 movement의 plan·actual roll-forward와 차이

| 사용자 의도 | route |
|---|---|
| 계획·예산·forecast·런웨이·자금소요 자체를 만든다 | `startup-finance` |
| 미래 가격 변경의 수요·이익을 예측한다 | `pricing` |
| BOM·노무·API·간접비·표준원가를 상세 구축한다 | `costing` |
| 실제 CFO 직접·간접법과 기말 현금을 계산·검산한다 | `fsa --solve cashflow` |
| 동결된 plan과 actual의 매출·영업이익·현금 차이를 설명한다 | `operating-variance` |

## 입력 확인 순서

다음 순서를 바꾸지 않는다. 앞 단계가 다르면 자동 보정하지 말고 올바른 비교 입력을 받는다.

1. **snapshot**: plan snapshot ID·승인 시점과 actual snapshot ID·마감 기준 시점을 고정한다.
2. **period**: 동일한 `[start, end)`·granularity·calendar인지 확인한다. 단월과 YTD를 섞지 않는다.
3. **scope**: entity·business unit·channel과 품목 정의가 같은지 확인한다.
4. **currency**: 양쪽이 같은 단일 통화인지 확인한다. 자동 환산하지 않는다.
5. **scale**: 원·천원·백만원 등 같은 배율인지 확인한다.
6. **evidence**: snapshot, 가격·수량, 손익 line mapping, cash category·sign의 근거를 연결한다.

그다음 선택한 `analysis_blocks`별로 묻는다.

- revenue: stable item ID, unit, `comparable|plan_only|actual_only`, 각 side의 quantity·unit price,
  non-quantity adjustment, 독립 source total
- operating profit: plan·actual source total, line ID, `income|expense|contra_revenue`, 각 side amount
- cash: plan·actual beginning/ending cash, stable movement ID, 외부 분류 category, signed amount

누락을 0으로 바꾸지 않는다. plan-only·actual-only는 presence 상태를 명시하고, refund·credit처럼
수량이 없는 매출은 PVM item에 섞지 않는다.

## 실행과 검산

1. catalog의 `input_contract.call_example`을 복사하고 `spec`만 실제 값으로 교체한다.
2. 단일 비교는 `fintool_run`의 `tool:"operating-variance"`로 실행한다.
3. 명시적 case override가 여러 개일 때만 공용 batch를 쓴다. `scenario`가 만든 case를 받더라도
   actual이나 plan snapshot을 엔진이 선택한 것으로 표현하지 않는다.
4. 성공 봉투에서 `schema_version`, `normalized_inputs`, `policies`, 세 reconciliation,
   `evidence_coverage`, warnings, 세 SHA-256 hash를 확인한다.
5. 숫자를 설명할 때 `raw_delta`는 항상 `actual - plan`임을 유지하고, effect는 산술 귀속일 뿐
   원인·책임 증명이 아니라고 밝힌다.

`arithmetically_reconciled`는 bridge 구성요소 합과 target delta가 허용오차 안에서 같다는 뜻이다.
`fully_explained`는 여기에 더해 `unexplained`가 0이고 필수 evidence가 모두 supported라는 뜻이다.
산술상 닫혀도 unsupported evidence가 있으면 완전히 설명된 것이 아니다. `unexplained`를 마지막
항목에 배부하거나 narrative로 숨기지 않는다.

해시는 `sha256:` 접두사를 포함한 전문을 인용한다.

- `inputs_hash`: evidence metadata까지 포함한 정규화 입력의 식별자
- `result_hash`: evidence·coverage·warning까지 포함한 전체 결과의 식별자
- `calculation_hash`: evidence 설명 metadata를 제외하고 schema·engine·정책·산술 결과를 묶은 식별자

## 금지사항

- reverse planner처럼 actual에서 계획·forecast를 역산하지 않는다.
- benchmark, prior actual, FX rate를 자동 조회·주입하지 않는다.
- variance로 원인·책임·통제가능성을 추정하지 않는다.
- `favorable`·`adverse`를 자동 판정하지 않는다.
- 현금 category나 손익 line의 법정 회계 적정성, 세금·세법을 판단하지 않는다.
- DSO·DIO·DPO나 AR·재고·AP에서 운전자본 cash effect를 역산하지 않는다.
- 관측 PVM을 미래 가격탄력성 evidence로 자동 넘기지 않는다.
- `unexplained`, unsupported evidence, snapshot cutoff와 revision을 생략하지 않는다.
