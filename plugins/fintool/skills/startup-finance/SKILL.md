---
name: startup-finance
description: fintool MCP로 스타트업 재무모델·시나리오·투자유치 자료를 만든다. 재무보고서, 피칭덱 재무, 런웨이, 자금소요, 캡테이블, LTV/CAC, bear/base/bull, 엑셀 워크북을 요청하면 이 스킬을 쓴다. startup-design·startup-pitch 등 스타트업 스킬 진행 중 재무 계산(Phase 7 재무, 검증 실험 판정, 피치 숫자)이 필요한 지점에서도 이 스킬을 쓴다. DCF·WACC·내재가치는 valuation, 상장사 재무제표·재무비율·부도확률은 financial-statements, 펀드 성과귀속·PE 펀드지표는 fund-performance, 최적 비중·VaR·거래비용은 portfolio-risk, 채권 가격·듀레이션은 fixed-income, 옵션 가격·내재변동성은 option으로 보낸다. 숫자는 추정하지 말고 fintool 봉투만 인용한다.
---

# Startup Finance

도구는 **원격 MCP**다. `fintool_catalog`로 스키마를 보고 `fintool_run`으로 실행한다. 로컬 바이너리를 설치하지 않는다.

이 스킬 범위: 인터뷰 → 드라이버 모델/유닛/시나리오/캡테이블 → `report`·워크북.
범위 밖: DCF·WACC·기업가치(`valuation`), 상장사 재무제표·재무비율·회계 정규화·부도확률(`financial-statements`), 펀드 성과귀속·PE 펀드지표(`fund-performance`), 포트폴리오 VaR·거래비용(`portfolio-risk`), 채권 프라이싱(`fixed-income`), 옵션 가격·내재변동성(`option`).

## 원칙

1. **LLM은 숫자를 만들지 않는다.** 수치는 JSON 봉투만 인용한다. 성장률을 반영한 월별 배열을 직접 계산해 넣는 것도 위반이다 — `growth`·`cohort` 스케줄을 쓴다.
2. **가정에 출처를 붙인다.** `source`: `창업자 인터뷰 YYYY-MM-DD`.
3. **HTML을 직접 쓰지 않는다.** `report` + `bundle.comments`.
4. **comments**는 인터뷰→모델→계산을 설명하되 새 숫자를 만들지 않는다.
5. 해시는 자르지 않는다. `business-plan`에는 해시가 없을 수 있다. 있는 것만 전문.

## 도구 선택

| 상황 | 도구 |
|------|------|
| 수익 메커니즘이 퍼널·티켓 구간·수수료 상한 등으로 이루어짐 | **`financial-model`** (드라이버 그래프) |
| 드라이버 3개 이하, 성장률 기반 단순 3개년 | `business-plan` |

**기본은 `financial-model`이다.** 사업의 수익이 "무엇 × 무엇 × 무엇"으로 만들어지는 이상, 그 곱셈을 모델 안에 두어야 시나리오가 의미를 갖는다. 곱셈을 채팅에서 하고 결과만 `revenue-base`에 넣으면 시나리오 축이 하나로 붕괴한다.

## 모드 선택 — cash / accrual

드라이버 그래프는 하나다. `mode`가 가르는 것은 **회계 관점뿐**이다.

| 상황 | 모드 |
|---|---|
| 초기 재무추정, "현금이 언제 마르나" | **cash** |
| 시나리오를 빠르게 여러 개 돌릴 때 | **cash** |
| 기초 대차대조표·회수조건을 모를 때 | **cash** |
| 투자자 실사·IR, 재무제표 3종이 필요 | **accrual** |
| 차입 조달 (은행이 재무제표를 본다) | **accrual** |
| 운전자본이 크게 도는 사업 (커머스 재고, B2B 장기 회수) | **accrual** |
| 세금·감가상각이 손익에 유의미 | **accrual** |

**기본은 cash다.** `mode`를 생략하면 cash다. accrual은 추가 입력을 요구하는데, **모르는 값을 0으로 채운 재무상태표는 정보가 없으면서 "재무제표가 나왔다"는 착시를 준다.** AR·재고·AP가 전부 0인 BS를 IR에 싣는 것이 그 착시의 결과다.

### 두 모드를 나란히 낼 때

같은 스펙을 `--mode`만 바꿔 두 번 돌릴 수 있다. accrual 전용 블록이 들어 있어도 cash 모드는 무시한다. 투자 검토 직전처럼 "현금 관점과 회계 관점을 같이 봐야 하는" 국면에서 쓴다.

**차이를 설명할 수 있어야 두 답을 나란히 낼 자격이 있다.** 차이의 원천은 **법인세·감가상각·운전자본 셋뿐**이고, 셋 다 없으면 기말현금이 정확히 일치해야 한다. 안 맞으면 설명이 아니라 스펙을 다시 본다. 각 원천이 어떻게 갈리는지는 `references/accrual-inputs.md`.

## 흐름

```
1. 인터뷰     → 수익 메커니즘을 드라이버 체인으로 (source 필수)
2. 모드 판정  → 기본 cash. accrual이면 추가 인터뷰
3. Base       → financial-model + unit-economics  → 턴 조각
4. 시나리오   → 드라이버 축으로 bear/base/bull, 미조달, oat  → 턴 조각
5. 캡테이블   → cap-table-simulate  → 턴 조각
6. 풀 리포트  → compose 없이 recipe 기본 / --workbook
7. 저장       → "모델을 저장할까요?" → fintool_save (스펙만)
다음 세션     → fintool_list → "저장된 모델 N개" → load 후 이어서
```

## 1단계 — 드라이버 발굴 인터뷰

고정 질문 목록을 읽지 말고 **돈이 들어오는 경로를 먼저 묻는다.**

> 고객이 돈을 내기까지 어떤 단계를 거치나요? 한 건당 얼마가 들어오고, 그게 무엇에 따라 달라지나요?

답을 드라이버 체인으로 옮긴다. 사업 유형별 체인 예시는 `references/business-models.md`에 있다 — 구독·성공보수·마켓플레이스·커머스·광고·하드웨어 6종. **템플릿을 강제하지 말고 참고만 한다.** 실제 사업은 대개 섞여 있다.

체인을 세운 뒤 각 마디의 값을 묻는다. 모르면 기본값을 제안하고 `source`에 `기본값 제안, 창업자 승인`.

공통으로 필요한 것: 통장 현금(`initial_cash`), 조달액(현금에 합산), 월 고정비, 인건비.

**여기까지가 cash 모드에 필요한 전부다.** 드라이버 + 기초현금이면 끝난다.

### accrual로 갈 때만 더 묻는다

모드 판정에서 accrual이 나왔을 때만, 그때 묻는다. **cash로 갈 사업에 아래를 묻지 않는다** — 답을 모르는 창업자에게 물어 0을 받아내면 그 0이 재무제표에 실린다.

| 묻는 것 | 블록 |
|---|---|
| 기초 대차대조표 — 현금 외 AR·재고·PP&E·AP·차입금·자본금·이익잉여금 | `opening_balance_sheet` |
| 회수·지급 조건 (매출채권 회수일, 재고 회전, 매입채무 지급일) | `working_capital` |
| 유형자산 내용연수, 기존 감가상각 | `ppe` |
| 차입 상환 계획·이자율 | `debt` |
| 법인세율·결손금 이월 | `tax` |
| 증자·배당 계획 | `equity` |
| 최소현금·리볼버 | `solver` |

**전부 선택 항목이다.** 안 주면 0으로 채우되 엔진이 `default_applied` 경고를 낸다.

```
경고: working_capital: 전부 0으로 채웠습니다. 매출·비용이 인식과 동시에
      현금으로 정산된다고 가정한 것입니다
```

**이 경고를 삼키지 말고 사용자에게 그대로 전달한다.** 자동으로 채운 0과 창업자가 확인한 0은 다르다. 블록별 기본값과 자동 균형 규칙은 `references/accrual-inputs.md`.

자본을 주지 않으면 자산과 같은 금액을 `contributed_capital`로 넣어 균형을 맞춘다. **반면 0을 명시하면 의도로 보고 불균형을 그대로 거부한다** — "모른다"와 "0이다"를 구분하기 위해서다.

## 2단계 — 드라이버 스펙

각 드라이버는 `id`·`label`·`value_type`·`unit`·`schedule`이다. 금액은 원(정수), 비율은 소수(`0.07` = 7%).

| schedule kind | 쓸 때 | 필드 |
|---|---|---|
| `monthly` | 전 기간 고정값 (수수료율·전환율 대부분) | `values: [0.3]` |
| `growth` | 복리 성장 | `base`, `monthly_growth` |
| `cohort` | 전월 잔량이 이월되는 기반 (구독 고객, 설치 대수) | `initial`, `retention`, 선택 `additions` |
| `recurring` | 구간 상수 | `value`, 선택 `start`·`end` |
| `one_time` | 단발 | `value`, `month` |

**`start`·`end`·`month`는 `"2026-10"` 같은 `"YYYY-MM"` 문자열이다.** `1`·`0` 같은 월 인덱스 숫자를
넣으면 계산 이전에 `invalid_json`으로 죽는다(`cannot unmarshal number into ... schedule.start`).
모델 기간(`start_month`부터 `months`개월) 안의 실제 월이어야 하고, 밖이면 `invalid_period`다.
`recurring`·`cohort`에서 `start`·`end`를 빼면 전 기간이다.

**kind마다 받는 키가 정해져 있다.** 표에 없는 키를 섞으면 `invalid_schedule`이다 —
`monthly`에 `start`를 붙이거나 `growth`에 `value`를 붙이는 식이 그렇다.

**성장·잔존을 직접 계산해 `monthly.values`에 36개 값으로 넣지 않는다.** `growth`·`cohort`가 그걸 하려고 있다. 배열을 손으로 만드는 순간 원칙 1이 깨진다.

## 3단계 — line item 수식

드라이버를 조합해 매출·원가·비용을 **선언**한다. 채팅에서 곱해 넣지 않는다.

expression은 `ref` | `literal` | `op` 중 하나이고 `op`의 `args`는 정확히 2개다.

| op | 용도 |
|---|---|
| `add` `subtract` | 같은 type·unit끼리만 |
| `multiply` | ratio × 숫자, money × count |
| `divide` | 숫자 ÷ ratio, 동일타입끼리는 ratio |
| `min` `max` | **수수료 상한·최소수수료** |

op 이름은 위 여섯 개 그대로다. `mul`·`sub` 같은 줄임말은 거부된다.
`id`는 바깥 expression에만 붙이고 **`args` 안쪽 항에는 붙이지 않는다.**

### 매출 × 원가율 — 최소 스펙

`ratio` 드라이버와 중첩 수식이 함께 나오는 가장 짧은 형태다. 그대로 실행된다.

```json
{"version":"financial-model/v3","mode":"cash","start_month":"2026-09","months":12,"currency":"KRW",
 "initial_cash":{"type":"money","unit":"KRW","value":600000000},
 "drivers":[
   {"id":"rev","label":"매출","value_type":"money","unit":"KRW",
    "schedule":{"kind":"growth","base":30000000,"monthly_growth":0.3}},
   {"id":"cogs_rate","label":"원가율","value_type":"ratio","unit":"decimal",
    "schedule":{"kind":"monthly","values":[0.2]}}],
 "line_items":[
   {"id":"revenue","label":"매출","category":"revenue","subtype":"transaction","frequency":"recurring",
    "value_type":"money","unit":"KRW","expression":{"id":"e.revenue","ref":"rev"},"statement_role":"revenue"},
   {"id":"cogs","label":"매출원가","category":"cost","subtype":"variable","frequency":"recurring",
    "value_type":"money","unit":"KRW",
    "expression":{"id":"e.cogs","op":"multiply","args":[{"ref":"rev"},{"ref":"cogs_rate"}]},
    "statement_role":"cogs"}]}
```

**`value_type`과 `unit`은 짝이다.** `money`면 `unit`이 통화(`KRW`), `ratio`면 `unit`이 반드시
`decimal`이다. `"unit":"ratio"`는 거부된다 — `ratio`는 unit이 아니라 value_type이다.

### 구간 비용과 단발 지출 — 최소 스펙

`recurring`·`one_time`이 `"YYYY-MM"` 월과 함께 나오는 가장 짧은 형태다. 그대로 실행된다.

```json
{"version":"financial-model/v3","mode":"cash","start_month":"2026-09","months":12,"currency":"KRW",
 "initial_cash":{"type":"money","unit":"KRW","value":600000000},
 "drivers":[
   {"id":"rent","label":"임대료","value_type":"money","unit":"KRW",
    "schedule":{"kind":"recurring","value":5000000,"start":"2026-10","end":"2027-03"}},
   {"id":"setup","label":"초기 구축비","value_type":"money","unit":"KRW",
    "schedule":{"kind":"one_time","value":80000000,"month":"2026-09"}}],
 "line_items":[
   {"id":"opex","label":"임대료","category":"cost","subtype":"fixed","frequency":"recurring",
    "value_type":"money","unit":"KRW","expression":{"id":"e.opex","ref":"rent"},"statement_role":"opex"},
   {"id":"capex","label":"초기 구축비","category":"investment","subtype":"capex","frequency":"one_time",
    "period":{"start":"2026-09","end":"2026-09"},
    "value_type":"money","unit":"KRW","expression":{"id":"e.capex","ref":"setup"},"statement_role":"capex"}]}
```

**드라이버의 `schedule`과 line item의 `period`는 별개다.** `frequency:"one_time"` line item은
`period:{start, end}`가 **필수**이고 두 값이 같은 월이어야 한다(`invalid_period`).
`frequency:"recurring"`이면 `period`는 선택이고, 빼면 전 기간이다.

`min`/`max`가 저액 구간 역마진을 계산 가능하게 만든다. 100만원 환급에 30% 수수료지만 상한 20만원이면 `min`, 최소수수료 10만원이면 `max`다. 이 계산이 없으면 역마진을 말로만 지적하게 된다.

### 퍼널 중간 단계는 `metric`으로 노출한다

`category: "metric"`은 어떤 재무 집계에도 들어가지 않는다. 조회→판정→신청→인용 각 단계를 `metric` line item으로 두면 월별 건수가 결과에 나오고, 워크북에도 행이 생긴다.

퍼널 전체를 매출 하나의 중첩 expression에 밀어넣으면 계산은 맞지만 **중간 건수가 결과에서 사라져** 다시 손으로 계산하게 된다. 원칙 1 위반 경로다.

`category`는 `revenue`·`cost`·`workforce`·`investment`·`funding`·`metric` 6개. `subtype`은 자유 문자열이므로 `success_fee`·`take_rate`·`consumable`처럼 의미대로 쓴다.

### statement_role — 회계 역할을 명시한다

`statement_role`은 선택 필드지만 **원가와 판관비를 구분해야 하는 사업에서는 반드시 준다.** 값은 `revenue`·`cogs`·`opex`·`capex`·`equity_raise`·`debt_draw`·`debt_repay`·`metric`.

모드가 처리를 가르는 대표가 `capex`다.

| `statement_role` | cash 모드 | accrual 모드 |
|---|---|---|
| `revenue` | 영업현금 유입 | 손익 매출 |
| `cogs` | 영업현금 유출 | **매출원가 — 매출총이익을 만든다** |
| `opex` | 영업현금 유출 | 판매관리비 |
| `capex` | **투자현금 유출, 손익 제외** | **자산화 → 정액 감가상각** |
| `equity_raise` | 재무현금 유입 | 자본 증자 |
| `debt_draw` / `debt_repay` | 재무현금 유입/유출 | 차입금 증감 + 이자 |
| `metric` | 집계 제외, 결과에 노출 | 동일 |

**생략하면 category에서 추론하는데, `cost`는 전부 `opex`로 간다.** 매출원가가 판관비로 분류되어 **매출총이익률이 100%로 나온다.** 순이익과 기말현금은 맞기 때문에 총액만 보면 넘어가고, 그 100%가 IR 자료에 실린다.

엔진이 이 상황을 진단으로 낸다.

```
warning statement_role_inferred: statement_role을 주지 않아 cost line item을 전부
  opex로 추론했습니다...
warning zero_cogs: 매출원가가 0이라 매출총이익률이 100%로 나옵니다...
```

커머스·하드웨어처럼 매출에 연동되는 원가가 있는 사업은 `statement_role: "cogs"`를 명시한다. 같은 경제를 두 방식으로 돌린 실측(`comparison/explicit-roles` vs `inferred-roles`):

```
명시  revenue 240,000,000  cogs 108,000,000  매출총이익률  55.0%
추론  revenue 240,000,000  cogs           0  매출총이익률 100.0%
```

**순이익과 기말현금은 두 경우가 완전히 같다.** 총액이 맞으니 검토에서 걸러지지 않는다.

## 시나리오

**드라이버 축으로 짠다.** 판정률×평균환급액, 전환율×티켓 같은 2축 그리드가 단일 매출 축보다 의사결정에 쓸모 있다.

- named bear/base/bull은 `scenario --mode corners`가 아니다. batch cases를 직접 짠다.
- 민감도는 `scenario --mode oat`로 상위 드라이버 ±30%.
- 미조달 케이스: `initial_cash` = 현재현금만. 조달 케이스: 현재현금 + 조달액.
- **전 시나리오가 같은 부호(전부 흑자/전부 적자)면 가정을 다시 본다.** 시나리오가 아니라 상수를 흔든 것이다.

## 유닛 이코노믹스

| 사업 형태 | 모델 | 주의 |
|---|---|---|
| 반복 결제 | `unit-economics --model subscription` | `arpa`·`arpa-period`·`gross-margin`·`churn-rate`·`churn-period`·`cac` 전부 필수, period 동일 |
| 단발 거래·성공보수 | `unit-economics --model transaction` | **LTV를 쓰지 않는다.** 재이용이 없으면 건당 공헌이익과 손익분기 건수가 답이다 |

단발 환급에 구독 LTV(연 재이용률)를 붙이지 않는다. 재이용이 실제로 있으면 창업자에게 근거를 묻고 `source`에 남긴다.

## 산출물

### 턴마다 조각

계산 도구가 성공한 턴에는 **그 자리에서** 조각 HTML을 낸다. 마지막에 몰아서 풀 리포트만 만들지 않는다.
모델이 도구를 안 부르고 "완성"이라고 선언하는 실패(#4997)를, 조각이 없으면 그 턴에서 드러낸다.

경로: `fintool_run` + `tool=report`. **`fintool_report`에 compose를 넣지 않는다.**

최소 구성은 항상 `header` + 그 턴 부품 + `assumption-ledger`. 부품만 뽑으면 거절된다.

| 이 턴의 도구 | 조각 부품 |
|---|---|
| `business-plan` (`base`) | `pl-cash-table` |
| `business-plan` (`unfunded`) | `runway-track` |
| `business-plan` (`bear`/`bull`) | `scenario-table` |
| `financial-model` | `fm-summary-table` |
| `financial-model --workbook` | `workbook-table` |
| `unit-economics` | `unit-econ-table` |
| `cap-table-simulate` | `cap-bar` |
| `business-plan --batch` | `batch-summary` |

전체 매핑·동반 부품·제외 목록은 저장소 `docs/tools/report-fragments.md`.

실행 가능한 조각 호출이다. 숫자만 바꿔 그대로 쓴다.

```json
{"tool":"report","flags":{"recipe":"finance-report","spec":{"meta":{"title":"예시","date":"2026-08-23","raise":500000000},"base":{"ok":true,"tool":"business-plan","data":{"rows":[{"year":1,"revenue":1200000000,"ebitda":-540000000,"cash_end":2881095890}]}}},"compose":{"recipe":"finance-report","components":["header","pl-cash-table","assumption-ledger"]}}}
```

`out`은 생략한다. 배달은 Worker가 고른다. `data.html`이 오면 한 글자도 바꾸지 말고 HTML 아티팩트로 낸다. `data.url`이 오면 그 링크와 `data.summary.figures`를 인용한다.

#### 조각을 내지 않는 때

1. 같은 부품을 이미 냈고 값이 안 바뀐 재계산
2. 도구가 실패한 턴 (입력 오류 재시도 포함)
3. 계산이 없는 인터뷰·가정 수집 턴
4. `fintool_catalog` / `fintool_save` / `fintool_load` / `fintool_list`
5. 마지막 풀 리포트를 내는 턴 — 조각을 겹쳐 내지 않는다
6. 서사·입력 표만 갱신한 턴. 서사 단독 조각은 빈 리포트 가드가 거절한다

### 풀 리포트

모든 계산이 끝나면 봉투를 `bundle.json`으로 모아 compose 없이 한 번 더 낸다.

```json
{"tool":"report","flags":{"recipe":"finance-report","spec":{"meta":{"title":"예시","date":"2026-08-23","raise":500000000},"base":{"ok":true,"tool":"business-plan","data":{"rows":[{"year":1,"revenue":1200000000,"ebitda":-540000000,"cash_end":2881095890}]}}}}}
```

### 봉투 키는 도구마다 정해져 있다

`spec`의 각 키는 **받는 도구가 고정**이다. 이름이 다르면 그 봉투는 조용히 무시되고
`바인딩할 계산 결과가 없습니다`만 돌아온다. `base`에 `financial-model` 결과를 넣는 실수가 가장 흔하다.

| 키 | 받는 도구 |
|---|---|
| `base` `unfunded` `bull` `bear` | `business-plan` |
| `financial_model` | `financial-model` |
| `unit_econ` | `unit-economics` |
| `cap_table` | `cap-table-simulate` |
| `capital_budget` | `capital-budget` |
| `fsa` `amort` `wacc` `dcf` `tvm` `npv` `rate_convert` | 같은 이름의 도구 |

`montecarlo`처럼 위 표에 없는 도구의 봉투는 받는 자리가 없다. 넣어도 그려지지 않는다.

봉투 밖 키는 `meta`, `assumptions[]`, `use_of_funds[]`, `traction[]`, `market[]`,
`milestones[]`, `competition[]`, `narrative{}`, `comments{}`다. 각 봉투는 `{ok, tool, data}`를
**도구가 돌려준 그대로** 넣는다. `data` 안을 손으로 고쳐 쓰면 언마샬에서 거부된다.

`meta.raise`는 **원 단위 숫자**다. 통화는 KRW로 고정이고 `meta`에 `currency` 필드는 없다.

```json
{"meta":{"title":"시리즈 A","date":"2026-08-25","subject":"B2B SaaS","raise":500000000},
 "financial_model": {"ok":true,"tool":"financial-model","data":{}}}
```

`"5억 원"`이나 `{"amount":500000000,"currency":"KRW"}`는 `invalid_type` at `$.meta.raise`로 거부된다.

| 레시피 | 용도 |
|--------|------|
| `finance-report` | 내부 재무 뷰. driver-trace 포함 |
| `pitch-deck` | IR. 서사·시장·Ask |
| `investor-update` | 기존 투자자 KPI·런웨이 |

`comments` 키는 부품 ID다. `pl-cash-table`, `driver-trace`, `ask-block`, `runway-track`.
조합 명세(`compose`)는 부품 ID 목록만. HTML·JSON 경로·숫자 리터럴 금지.

### HTML 회수 — 원격 MCP

`out`을 주면 HTML은 파일로만 나간다. **원격 MCP에서는 그 파일을 회수할 수 없다.** 원격이면 `out`을 생략한다.

**배달은 Worker가 고른다. 스킬이 봉투/링크를 고르지 않는다.**

| 응답 | 언제 | 할 일 |
|---|---|---|
| `data.html` | 대화 중 조각 (`compose` 있음, `workbook-table` 없음, 16 KiB 이하) | 한 글자도 바꾸지 말고 HTML 아티팩트로 낸다 |
| `data.url` + `data.summary` | 풀 리포트, `workbook-table`, 16 KiB 초과 | URL을 인용하고 `summary.figures`로 숫자를 말한다. `data.html`을 아티팩트로 다시 만들지 않는다 |

`summary.figures` 키는 `raise`, `revenue`, `ebitda`, `cash_end`, `runway_months`, `peak_funding_need`, `ltv`, `ltv_cac`, `post_money`다. 없는 값은 생략된다. 링크만 주면 모델이 결과를 말할 수 없어서 요약이 같이 온다.

로컬에서 파일이 필요하면 `out`을 준다. 렌더 결과는 같다.

### 공개 공유 — 사용자 명시 요청만

원격 리포트 링크는 **기본 비공개**다. 공개 링크로 바꾸는 것은 사용자의 재무제표를 인터넷에 올리는 일이다.

- **"공유해줘", "공개 링크로 바꿔줘"**처럼 사용자가 명시적으로 요청했을 때만 `fintool_report_share`를 부른다.
- 모델이 먼저 제안하거나, 확인 없이 공유하지 않는다.
- 호출 전에(또는 응답 `notice`를 그대로) 무엇을 공개하는지 말한다: 월별 현금흐름·손익, 캡테이블, 가정·코멘트 등 리포트 전체. 검색엔진에는 올리지 않는다(`noindex`).
- `action=share` — 별도 공개 키를 발급한다. 원래 `data.url`의 비공개 id는 타인이 열 수 없다.
- `action=unshare` — 공개를 해제한다. 옛 공개 주소는 404가 된다.
- `action=revoke` — 폐기 페이지만 남긴다.
- 증명서 `fintool_share`와 다르다. 리포트는 회수할 수 있다.

### 워크북

사용자가 **가정을 직접 바꿔볼 수 있는 스프레드시트**를 원하면 `financial-model`에 `--workbook`을 준다. 값이 아니라 가정 셀과 그것을 참조하는 수식이 나오므로, 받는 쪽에서 셀 하나를 바꾸면 전체가 재계산된다.

시트 구성은 모드에 따라 다르다. 레이아웃은 공통으로 `A=label B=id C=unit D·E·F=파라미터 G~=월`이다.

| 모드 | 시트 |
|---|---|
| cash | `Assumptions` · `Model` · `Summary` |
| accrual | `Assumptions` · `Model` · `Inputs` · `IS` · `BS` · `CF` |

accrual 워크북의 `BS`에는 `balance_check` 행이 있다 — `ROUND(총자산 − 총부채및자본, 2)`. **사용자가 가정을 바꿔도 이 행이 0이면 대차가 Excel 안에서 여전히 맞는다는 뜻이다.**

### xlsx로 전사한다

봉투는 시트·셀별 **수식 문자열**이다. 엔진은 xlsx 바이너리를 만들지 않으므로 파일로 옮기는 것은 이쪽 일이다. Claude Code·Codex는 openpyxl로 쓰고, Desktop·웹은 xlsx 스킬의 코드 실행 환경에서 **같은 openpyxl 코드**를 돌린다. 환경이 가르는 것은 전달 경로뿐이다.

**설계하지 않고 전사만 한다.** 지켜야 할 것 다섯 가지다.

1. **수식은 `formulas` 원문 그대로.** `=`로 시작하면 수식, 아니면 리터럴, **빈 문자열이면 셀을 비운다.** 시트 참조(`Assumptions!G$3`)를 손대지 않는다.
2. **수식을 읽고 다시 타이핑하지 않는다.** accrual 감가상각 수식은 36개월 실측 **2,358자**다. 스크립트가 JSON을 읽어 셀에 넣는다.
3. **열은 `layout.month_columns[i]`를 그대로 읽는다.** 직접 계산하면 21번째 달 `Z→AA`에서 어긋나는데, 수식 안 참조는 `AA`를 가리키므로 **`#REF!`도 안 뜨고 값만 조용히 틀린다.**
4. **시트 순서·행 번호를 바꾸지 않는다.** 수식이 절대행을 참조한다. 정렬·행 삽입 전부 금지다.
5. **값 셀은 `values` 캐시값이 아니라 수식을 쓴다.** 값을 써 넣으면 가정을 바꿔도 재계산되지 않아 워크북이 아니라 표가 된다.

**파일을 넘기기 전에 다시 열어 대조한다.** openpyxl로 재열어 수식 셀 수 = JSON 수식 수, 수식 문자열 원문 일치, 빈 셀·파라미터·서식을 본다. **불일치가 하나라도 있으면 파일을 제공하지 말고 먼저 보고한다** — 수식을 짐작해 고쳐 쓰는 순간 전사가 아니라 설계가 된다.

전사·검증 스크립트 전문과 오탐 두 가지(빈 문자열 수식 468셀, 날짜 서식 셀의 `datetime` 되돌림)는 `references/workbook-transcribe.md`.

파일명은 `<회사>-<모델>-<YYYYMMDD>.xlsx`다. 계산 해시는 자르지 말고 **한 번만** 인용한다. **워크북 안의 수식을 채팅에서 설명하거나 요약하지 않는다** — 설명은 워크북 A·B열에 이미 있다.

### 리볼버를 켜면 워크북이 안 나온다

`solver.revolver_enabled: true`면 `supported: false`와 이유가 오고 시트는 비어 있다. 이자↔차입잔액↔현금이 순환 참조를 이루는데 Excel은 반복 계산이 기본으로 꺼져 있기 때문이다.

**사용자가 워크북을 원한다고 밝혔으면 리볼버를 켜기 전에 이 제약을 먼저 알린다.** 다 돌리고 나서 "워크북은 안 됩니다"라고 하지 않는다.

재무 리포트만으로 충분한지, 조작 가능한 워크북이 필요한지 **먼저 묻는다.** 둘은 다른 산출물이다. 계약 전문은 `docs/decisions/workbook-contract.md`.

## 저장과 재개

Desktop 원격 MCP에는 파일시스템이 없다. 세션이 끝나면 스펙이 사라진다. "지난번 모델에 시나리오 추가해줘"는 서버에 스펙이 있을 때만 된다.

저장 도구는 **신원이 확인된 연결에서만** 보인다. `fintool_save` · `fintool_load` · `fintool_list`. 카탈로그에 없거나 `identity_required`·`store_unavailable`이면 저장하지 않은 것이다. 로컬 파일로 흉내 내지 않는다.

### 이 세션이 끝날 때

모델이 한 번이라도 돌아갔으면 **저장을 제안한다.** 숫자를 새로 만들지 말고 묻기만 한다.

> 이 모델을 저장할까요? 다음 세션에서 "지난번 모델에 시나리오 추가"가 됩니다.

동의하면 `fintool_save`에 **계산에 넣은 스펙**을 넣는다. 결과 봉투·HTML·워크북은 저장하지 않는다.

```
fintool_save name=아크메랩스-base spec=<financial-model 스펙> note="창업자 인터뷰 2026-08-25 base"
```

이름: `{회사}-{역할}`. 64자, 대소문자 구분 없음. 사용자가 이름을 주면 그걸 쓴다.

같은 이름에 다시 저장하려면 `overwrite: true`다. 기본은 `name_conflict`로 거절한다. 덮어쓸지 새 이름인지는 묻는다.

### 다음 세션이 시작될 때

재무 요청이 오면 **계산하기 전에** `fintool_list`를 부른다.

- `count`가 0이면 저장본이 없다고 말하고 인터뷰부터 한다.
- `count`가 1 이상이면 이름·head seq·갱신 시각을 보여 주고 제안한다.

> 저장된 모델 N개가 있습니다. 이어서 쓸까요?

"지난번 모델에 시나리오 추가"처럼 재개가 분명하면 묻지 않고 고른다. 하나면 그걸 쓰고, 여러 개면 이름을 고르게 한다.

```
fintool_load name=아크메랩스-base
```

받은 `data.spec`을 **한 글자도 바꾸지 말고** `financial-model`에 다시 넣는다. 시나리오는 그 스펙의 드라이버를 축으로 짠다. 채팅에서 성장률을 다시 계산해 넣지 않는다.

같은 이름에 새 버전을 남기려면 `overwrite: true`. 시나리오를 다른 이름으로 가르려면 `fintool_branch from=아크메랩스-base name=아크메랩스-bull`. 드라이버 하나만 고치려면 `fintool_patch`.

`list`의 `models[]`에는 스펙이 없다. 내용을 보려면 `load`다.

### 실패

| kind | 할 일 |
|---|---|
| `identity_required` | 로그인된 커넥터가 아니다. 저장·재개 불가라고 알린다 |
| `store_unavailable` | 원격 저장소가 닫혀 있다. 꾸며서 저장했다고 하지 않는다 |
| `not_found` | 그 이름(그 사용자)에 없다. 다른 사용자 모델은 원래 안 보인다 |
| `name_conflict` | overwrite 또는 새 이름을 묻는다 |
| `quota_exceeded` | 50개. 지우거나 기존 이름에 버전을 추가한다 |

해시는 자르지 않는다. 저장·load 봉투의 `hash`를 한 번 인용한다.

## 카탈로그 호출 줄이기

`fintool_catalog`는 스펙 조회일 뿐 계산이 아니다. 계측에서 전체 도구 호출의 31%가 카탈로그였다.

- 이 문서에 **호출 예시가 있는 도구는 카탈로그를 부르지 않는다.** 예시를 그대로 쓰고 숫자만 바꾼다.
- 예시가 없는 도구만 `fintool_catalog {"tool":"<이름>"}`으로 그 도구 하나를 받는다. 인자 없는 전체 목록은 어떤 도구가 있는지 모를 때만 쓴다.
- 호출이 실패하면 오류 봉투가 유효 플래그 목록과 `input_contract`를 함께 준다. 그것을 보고 고치면 되고 카탈로그를 따로 부를 필요가 없다.

## 호출 함정
- **금액을 원 단위 정수로 바꿀 때 자릿수를 다시 센다.** "5,000억"은 `500000000000`(0이 11개)이다. 엔진은 단위를 검증하지 못하고 그대로 계산한다.
- **저장은 스펙만.** 결과 봉투를 `fintool_save`에 넣지 않는다. 도구가 없거나 `store_unavailable`이면 로컬 파일로 대신 저장했다고 말하지 않는다. 다음 세션에 숫자를 다시 보려면 `load` 후 재실행한다 — 결정론적이라 `calculation_hash`는 같다.
- **같은 세션 안에서는 `run_id`를 쓴다.** `fintool_run` 성공 봉투에 붙는 `run_id`를 `report` 번들의 봉투 자리에 `{"$ref":"run_..."}`로 넣으면 55~75KB 결과를 다시 타이핑하지 않는다(보관 1시간). 이것이 원격 MCP에서 `financial-model` 기반 리포트를 내는 유일한 길이다.
- 다음 세션에서 저장본을 짐작하지 않는다. `fintool_list`가 말한 이름만 `load`한다.


- `fintool_run`의 `spec`은 **문자열**로 직렬화한다.
- 기본 플래그는 kebab-case. batch `params`는 **snake_case**, 스칼라만.
- `months`는 12~60이다.
- `mode`를 생략하면 cash다. `--mode`는 스펙의 `mode`를 덮어쓴다. `cash`·`accrual` 외의 값은 거부된다.
- **`--summary`는 두 모드 모두 붙는다.** accrual은 월별 핵심 5개 + `summary`로, cash는 월별 현금 5개(매출·비용·영업손익·순현금·기말현금) + 라인아이템별 연간 집계 + `runway`로 줄인다. `calculation_hash`는 전체 결과와 같으므로 축약본을 그대로 인용해도 된다. 36개월 모델은 이것 없이는 도구 결과 한도를 넘긴다.
- **`--scenarios`로 BEAR/BULL을 한 번에 낸다.** `[{"name":"bear","overrides":{"drivers.<id>.schedule.monthly_growth":0.03}}]`. 스펙을 시나리오마다 다시 보내지 않는다. 응답은 이미 축약 단위라 `--summary`와 함께 쓰지 않는다.
- **`cohort`·`growth` 스케줄의 파라미터에 참조를 쓴다.** 유입이 `가입 × 전환 × 보유율`이면 그 곱을 `metric` line item으로 두고 `"additions": {"ref": "m_new"}`로 가리킨다. 곱을 손으로 계산해 상수로 박으면 원칙 1(LLM은 숫자를 만들지 않는다)이 깨진다.
- **기존 `financial-model/v1`·`v2` 스펙도 그대로 받는다.** 다만 그 두 버전에 `--mode`를 주면 거부된다 — v1은 cash, v2는 accrual로 고정이다. 모드를 바꿔 돌리려면 스펙을 새 형식으로 옮긴다.
- expression은 **같은 월 값만** 참조한다. 전월 참조는 `expression_cycle`로 거부된다 — 시간 재귀는 `growth`·`cohort` 스케줄로 푼다.
- 재무 line item의 `value_type`은 `money`다. `count`·`ratio`는 `metric`에서만.
- `business-plan`을 쓸 때: **조달금을 `equity_raise` params에 넣지 않는다.** 전 연도 broadcast된다. `nol-carryforward: true`를 기본으로 켠다.
- cap-table `round`는 pre-money가 아니라 `price_per_share`. SAFE가 있으면 2~3회 맞춰 수렴한다.
- `business-plan.free_cash_flow`를 `dcf`에 넣지 않는다. 그건 valuation 스킬이다.

## startup-skill 연결

같은 플러그인의 서드파티 스킬(`startup-design`·`startup-competitors`·`startup-positioning`·`startup-pitch`)이
계산 지점에 도달하면 산출물 형식은 그 스킬을 따르되 **숫자는 전부 이 규약으로 fintool에서 낸다**.

| 트리거 | fintool 호출 |
|--------|--------------|
| Phase 7 프로젝션 (M1–12 / Y1–3) | `financial-model` (드라이버, 기본 cash) / `business-plan` (단순 연간) |
| Phase 7 재무제표 3종·BS 균형 | `financial-model --mode accrual` |
| Phase 7 민감도 ±30% | `scenario --mode oat` |
| Phase 7 conservative/base/optimistic | named bear/base/bull batch (`--mode corners` 아님) |
| Phase 7 CAC·LTV·churn | `unit-economics --model subscription` |
| Phase 7 break-even | `unit-economics --model transaction` |
| Phase 7 runway·자금소요 | `financial-model` 미조달/조달 케이스 + `montecarlo --failure-rate` |
| Phase 8 실험 pass/fail 판정 | `stats --mode test --test independence --method fisher-exact` |
| pitch Ask·런웨이 | `financial-model` 자금소요액 |
| pitch 희석 | `cap-table-simulate` |
| 덱·IR 산출물 | `report` (`pitch-deck` / `investor-update`) |
| 사용자가 직접 조작할 모델 | `financial-model --workbook` |

- `[Assumption]`/`[Estimate]` 라벨은 `assumptions[].source`에 출처로 남긴다. 고객 인터뷰 게이트 통과 전 재무는 Stage A(가정 기반)임을 명시한다.
- TAM/SAM/SOM·RICE 같은 자명한 산술은 fintool 없이 계산하고 가정 출처를 명시한다.
- 서드파티 SKILL.md는 수정하지 않는다(업데이트 시 덮어써짐). 유지보수는 이 섹션만 한다.

## 확장

가격 실험: `pricing`. 밸류: `valuation` 스킬.

| 참고 | 내용 |
|---|---|
| `references/business-models.md` | 사업 유형별 드라이버 체인 6종 + 유형별 모드 권고 |
| `references/accrual-inputs.md` | accrual 전용 블록 7종, 기본값·자동 균형 규칙, 불변식 |
| `references/workbook-transcribe.md` | `--workbook` 봉투 → xlsx 전사·자체 검증 스크립트, 대조 오탐 2종 |
