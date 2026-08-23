---
name: startup-finance
description: fintool MCP로 스타트업 재무모델·시나리오·투자유치 자료를 만든다. 재무보고서, 피칭덱 재무, 런웨이, 자금소요, 캡테이블, LTV/CAC, bear/base/bull, 엑셀 워크북을 요청하면 이 스킬을 쓴다. startup-design·startup-pitch 등 스타트업 스킬 진행 중 재무 계산(Phase 7 재무, 검증 실험 판정, 피치 숫자)이 필요한 지점에서도 이 스킬을 쓴다. DCF·WACC·내재가치는 valuation 스킬로 보낸다. 숫자는 추정하지 말고 fintool 봉투만 인용한다.
---

# Startup Finance

도구는 **원격 MCP**다. `fintool_catalog`로 스키마를 보고 `fintool_run`으로 실행한다. 로컬 바이너리를 설치하지 않는다.

이 스킬 범위: 인터뷰 → 드라이버 모델/유닛/시나리오/캡테이블 → `report`·워크북.
범위 밖: DCF·WACC·기업가치(`valuation`), 포트폴리오 VaR, 채권 프라이싱.

## 원칙

1. **LLM은 숫자를 만들지 않는다.** 수치는 JSON 봉투만 인용한다. 성장률을 반영한 월별 배열을 직접 계산해 넣는 것도 위반이다 — `growth`·`cohort` 스케줄을 쓴다.
2. **가정에 출처를 붙인다.** `source`: `창업자 인터뷰 YYYY-MM-DD`.
3. **HTML을 직접 쓰지 않는다.** `report` + `bundle.comments`.
4. **comments**는 인터뷰→모델→계산을 설명하되 새 숫자를 만들지 않는다.
5. 해시는 자르지 않는다. `business-plan`에는 해시가 없을 수 있다. 있는 것만 전문.

## 도구 선택

| 상황 | 도구 |
|------|------|
| 수익 메커니즘이 퍼널·티켓 구간·수수료 상한 등으로 이루어짐 | **`financial-model` v1** (드라이버 그래프) |
| 드라이버 3개 이하, 성장률 기반 단순 3개년 | `business-plan` |
| 대차대조표·부채·운전자본 정합성이 필요 | `financial-model` v2 |

**기본은 v1이다.** 사업의 수익이 "무엇 × 무엇 × 무엇"으로 만들어지는 이상, 그 곱셈을 모델 안에 두어야 시나리오가 의미를 갖는다. 곱셈을 채팅에서 하고 결과만 `revenue-base`에 넣으면 시나리오 축이 하나로 붕괴한다.

## 흐름

```
1. 인터뷰     → 수익 메커니즘을 드라이버 체인으로 (source 필수)
2. Base       → financial-model v1 + unit-economics
3. 시나리오   → 드라이버 축으로 bear/base/bull, 미조달, oat
4. 캡테이블   → cap-table-simulate
5. 산출물     → report(bundle.json + comments) / --workbook
```

## 1단계 — 드라이버 발굴 인터뷰

고정 질문 목록을 읽지 말고 **돈이 들어오는 경로를 먼저 묻는다.**

> 고객이 돈을 내기까지 어떤 단계를 거치나요? 한 건당 얼마가 들어오고, 그게 무엇에 따라 달라지나요?

답을 드라이버 체인으로 옮긴다. 사업 유형별 체인 예시는 `references/business-models.md`에 있다 — 구독·성공보수·마켓플레이스·커머스·광고·하드웨어 6종. **템플릿을 강제하지 말고 참고만 한다.** 실제 사업은 대개 섞여 있다.

체인을 세운 뒤 각 마디의 값을 묻는다. 모르면 기본값을 제안하고 `source`에 `기본값 제안, 창업자 승인`.

공통으로 필요한 것: 통장 현금(`initial_cash`), 조달액(현금에 합산), 월 고정비, 인건비.

## 2단계 — 드라이버 스펙

각 드라이버는 `id`·`label`·`value_type`·`unit`·`schedule`이다. 금액은 원(정수), 비율은 소수(`0.07` = 7%).

| schedule kind | 쓸 때 | 필드 |
|---|---|---|
| `monthly` | 전 기간 고정값 (수수료율·전환율 대부분) | `values: [0.3]` |
| `growth` | 복리 성장 | `base`, `monthly_growth` |
| `cohort` | 전월 잔량이 이월되는 기반 (구독 고객, 설치 대수) | `initial`, `retention`, 선택 `additions` |
| `recurring` | 구간 상수 | `value`, `start`·`end` |
| `one_time` | 단발 | `value`, `month` |

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

`min`/`max`가 저액 구간 역마진을 계산 가능하게 만든다. 100만원 환급에 30% 수수료지만 상한 20만원이면 `min`, 최소수수료 10만원이면 `max`다. 이 계산이 없으면 역마진을 말로만 지적하게 된다.

### 퍼널 중간 단계는 `metric`으로 노출한다

`category: "metric"`은 어떤 재무 집계에도 들어가지 않는다. 조회→판정→신청→인용 각 단계를 `metric` line item으로 두면 월별 건수가 결과에 나오고, 워크북에도 행이 생긴다.

퍼널 전체를 매출 하나의 중첩 expression에 밀어넣으면 계산은 맞지만 **중간 건수가 결과에서 사라져** 다시 손으로 계산하게 된다. 원칙 1 위반 경로다.

`category`는 `revenue`·`cost`·`workforce`·`investment`·`funding`·`metric` 6개. `subtype`은 자유 문자열이므로 `success_fee`·`take_rate`·`consumable`처럼 의미대로 쓴다.

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

봉투를 `bundle.json`으로 모은다. 형식은 `docs/examples/report/acme.bundle.json`.

```
fintool_run tool=report flags={recipe, spec, out}
```

| 레시피 | 용도 |
|--------|------|
| `finance-report` | 내부 재무 뷰. driver-trace 포함 |
| `pitch-deck` | IR. 서사·시장·Ask |
| `investor-update` | 기존 투자자 KPI·런웨이 |

`comments` 키는 부품 ID다. `pl-cash-table`, `driver-trace`, `ask-block`, `runway-track`.
조합 명세(`compose`)는 부품 ID 목록만. HTML·JSON 경로·숫자 리터럴 금지.

### 워크북

사용자가 **가정을 직접 바꿔볼 수 있는 스프레드시트**를 원하면 `financial-model`에 `--workbook`을 준다. 값이 아니라 가정 셀과 그것을 참조하는 수식이 나오므로, 받는 쪽에서 셀 하나를 바꾸면 전체가 재계산된다.

시트는 `Assumptions`·`Model`·`Summary` 3장, 레이아웃은 `A=label B=id C=unit D·E·F=파라미터 G~=월`이다. 계약 전문은 `docs/decisions/workbook-contract.md`.

재무 리포트만으로 충분한지, 조작 가능한 워크북이 필요한지 **먼저 묻는다.** 둘은 다른 산출물이다.

## 호출 함정

- `fintool_run`의 `spec`은 **문자열**로 직렬화한다.
- 기본 플래그는 kebab-case. batch `params`는 **snake_case**, 스칼라만.
- v1 `months`는 12~60이다.
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
| Phase 7 프로젝션 (M1–12 / Y1–3) | `financial-model` v1 (드라이버) / `business-plan` (단순 연간) |
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

가격 실험: `pricing`. 밸류: `valuation` 스킬. 사업 유형별 드라이버 체인: `references/business-models.md`.
