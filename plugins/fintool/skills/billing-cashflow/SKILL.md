---
name: billing-cashflow
description: fintool MCP로 청구서부터 수금·결제수수료·환불·정산·은행 현금까지 계산하고 AR·aging·DSO를 검산한다. 연간 선불 vs 월 후불, 청구 후 현금 입금시점, 수금 지연, 미수금/AR, DSO, 결제수수료, 환불 현금흐름 요청에 사용한다. MRR·ARR는 subscription-metrics, 전사 계획은 business-plan·financial-model, 데이터 기반 재무 예측 검증은 forecast-validation으로 보낸다.
---

# Billing Cashflow

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

명시적 계약·청구·수금 event를 `invoice -> collection -> funds in transit -> bank cash`로 연결한다.
원격 MCP의 `billing-cashflow`를 사용한다. 첫 실행 전에
`fintool_catalog {"tool":"billing-cashflow"}`로 현재 `input_contract.call_example`, flags,
handoff와 typed workflow ports를 확인한다. catalog 예시의 `spec`을 실제 값으로 교체해
`fintool_run`에 객체로 전달하고 문자열로 이중 직렬화하지 않는다.

수치는 성공 JSON 봉투에서만 인용한다. 필드·공식·workflow port는 카탈로그가 반환한 최신 계약과
성공 봉투의 `normalized_inputs`·`reconciliation`·`explanations`를 정본으로 삼는다.

## Trigger와 계산 경계

다음 요청을 이 스킬로 처리한다.

- 연간 선불과 월 후불의 은행 현금 유입 시점 비교
- 청구 후 며칠 뒤 실제 현금이 입금되는지, 수금 지연·부분수금·연체가 현금에 미치는 영향
- 미수금, AR roll-forward, aging, period-end DSO, realized collection days
- percent+fixed 결제수수료, credit note, 환불 현금유출, settlement lag
- 고객 결제 성공과 결제사업자 정산·은행 가용현금의 시점 차이

invoice는 revenue가 아니고, collection은 bank cash가 아니다. 다음 사건을 합치지 않는다.

```text
invoice --applied collection--> AR 감소
collection --fee/refund--> funds in transit
settlement --> bank cash
```

`mode:"actual"`은 실제 invoice·payment·settlement ledger를, `mode:"plan"`은 사용자가 승인한
fixed cadence·collection profile·fee·settlement 가정을 쓴다. actual을 plan으로 자동 승격하지
않는다.

## 첫 확인

빠진 값을 0이나 관행값으로 채우지 말고 다음을 확인한다.

1. 단일 ISO 4217 통화와 `minor_unit_exponent`, 분석기간·`as_of`·IANA timezone
2. actual인지 명시적으로 승인된 plan인지와 근거의 `as_of` 또는 유효기간
3. `count:1`인 월·분기·연 cadence, full service period, anchor, `advance|arrears`, invoice offset,
   calendar-day payment terms
4. actual invoice·payment allocation·credit note·refund 요청일·환불 현금 유출일·settlement 또는
   plan collection buckets
5. percent rate, fixed fee, cap/floor, fee basis, 거래별 rounding, refund fee 규칙과 유효기간
6. settlement lag·refund funding source와 opening AR·funds in transit·bank cash
7. source·assumption·derived·unsupported evidence가 계산 필드를 설명하는지

P0는 full period fixed cadence만 자동 생성한다. partial period, usage arrears, business-day calendar,
retry/dunning, tiered fee는 근사하지 않는다.

## 입력과 실행

catalog의 strict `billing-cashflow/v1` example을 출발점으로 쓴다. 모든 money는 major-unit float가
아니라 `amount_minor` 정수다. `minor_unit_exponent`가 2인 USD의 `10000`은 USD 100.00이고,
지수가 0인 KRW의 `10000`은 KRW 10,000이다.

- 명시적 zero는 실제 값이다. zero payment terms, fee, opening balance, refund는 유효하다.
- missing은 zero가 아니다. 필수 terms·fee·lag·evidence가 없으면 계산하지 않거나 입력 오류로
  처리한다.
- credit note와 refund 금액은 양수로 주고 event type이 경제적 부호를 결정한다.
- 같은 날짜 event는 `invoice -> credit_note -> writeoff -> collection -> processor_fee -> refund ->
  settlement` precedence와 stable ID 순서로 처리한다. 입력 배열 순서로 결과를 조정하지 않는다.
- `calculation_hash`는 계산 의미론과 evidence를 제외한 정규화 계산 입력을 묶는 canonical hash다.
  현재 v1에는 evidence 전용 hash가 없으므로 이 hash가 source·assumption 동일성까지 증명한다고
  설명하지 않는다.

여러 승인 case는 공용 `--batch`를 쓰고, 승인된 range를 전개할 때만 `scenario`를 연결한다.
여러 도구를 잇는 경우 typed workflow port의 currency, minor-unit exponent, period와 scope를
보존한다. 실제 명령은 도구 문서의 검증된 예제를 복사한다.

## 결과 승인

숫자를 설명하기 전에 다음 세 roll-forward와 검산 차이가 정확히 0 minor unit인지 확인한다.

```text
ending AR
  = opening AR + invoices - credit notes - applied collections - writeoffs

ending funds in transit
  = opening funds in transit + collections - processor fees
  - processor-funded refunds and fees - bank settlements

ending bank cash
  = opening bank cash + bank settlements
  - bank-funded refunds and fees
```

invoice total·payment allocation·aging 검산도 함께 확인한다. 성공 결과의 필수 difference가 0이
아니면 결과를 승인하지 않는다.

- `period_end_dso_days`는 기말 AR와 해당 기간 credit sales의 비율이다. credit sales가 0이면
  `null`과 explanation이어야 한다.
- `realized_collection_days`는 수금된 금액으로 가중한 invoice-to-paid 일수다. 미수 invoice는
  aging에 남으며 이 분모에 넣지 않는다.
- aging bucket 합은 ending AR와 같아야 한다.
- `unsupported`, missing evidence, conflict, warnings와 explanations를 0 또는 verified로 바꾸지 않는다.

결과를 전달할 때 currency·minor-unit exponent, mode·period·as-of, gross billings·collections,
fees·refunds·settlements, ending AR·funds in transit·bank cash, DSO·realized days·aging,
reconciliation의 `fully_explained`·`status`, evidence 배열, warnings, `calculation_hash`를 함께
제시한다.

## Routing과 handoff

| 사용자 의도 | route |
|---|---|
| invoice·AR·수금·fee·refund·settlement·bank cash | `billing-cashflow` |
| MRR·ARR·waterfall·GRR/NRR·subscription cohort | `subscription-metrics` |
| 장기 사업계획의 연매출·비용·연말 운전자본·자금소요 | `business-plan` |
| 월별 회사 전체 손익·재무상태·현금·런웨이 | `financial-model` |
| “매출은 있는데 현금이 부족”, 선후불·DSO·수수료·환불을 포함한 창업 재무 질문 | `startup-finance`가 이 스킬로 라우팅 |
| 관측 시계열의 통계적 예측·rolling-origin·coverage | `forecast-validation` |

handoff는 자동 회계 판단이 아니다.

- `subscription-metrics`의 recurring amount·interval·active period는 invoice schedule의 후보일 뿐,
  MRR를 invoice·recognized revenue·cash로 동일시하지 않는다.
- `business-plan`에 DSO scalar를 넘기면 invoice ledger와 월중 cash trough가 사라진다.
  `lossiness`가 명시된 planning approximation으로만 사용한다.
- `financial-model`로 minor를 major unit으로 바꿀 때 currency·exponent·rounding metadata를 남긴다.
  funds in transit 계정분류와 deferred revenue는 사용자 근거가 없으면 만들지 않는다.
- 현재 v1 handoff는 `cost_scope`를 자동 생성하지 않는다. processor fee·refund를 downstream gross
  margin이나 비용에서 이미 반영했는지 호출자가 명시한 cost scope로 확인한다. 범위가 겹치면
  이중차감하므로 handoff하지 않는다.

## 금지

- 목표 bank cash·DSO에서 필요한 청구·수금 event를 역산하는 reverse planner
- 업계 benchmark·목표치·provider 조건 또는 missing fee·refund·terms·lag를 채우는 숨은 기본값
- DSO scalar에서 미래 수금분포를 역산하거나 과거 event에서 지연·환불의 원인을 자동 인과추정
- actual ledger를 plan·forecast로 자동 승격
- 검증되지 않은 statistical forecast를 계획값으로 자동 승격. 예측 검증은 `forecast-validation`
- 세금 계산·세법 판단, revenue recognition, 법정 계정분류·공시 적정성 판단
- 자동 FX 또는 여러 통화 합산
- unsupported·unverified·null을 0 또는 확인된 값으로 표현

P0 밖 요청은 가장 가까운 값으로 근사하지 않는다. 표현할 수 없는 의미론, 필요한 upstream
evidence와 적절한 downstream route를 밝힌다.
