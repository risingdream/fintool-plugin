---
name: gtm-economics
description: fintool MCP로 실제·계획 GTM 지출과 동일 cohort 퍼널의 CPM·CPC·단계별 획득단가, paid·blended CAC, attributed·incremental ROAS, ROMI, 목표 고객·매출·파이프라인에 필요한 리드와 예산을 계산한다. 마케팅 비용, 광고 효율, 채널별 CAC, 퍼널 전환율, GTM 경제성, 필요 리드, 필요 예산과 marketing spend, channel CAC, funnel conversion, GTM economics 요청에 사용한다. paid media·PLG·sales-led·partnership·event motion을 지원한다. 채널 전략·첫 100명 계획은 startup-design, 확정 CAC의 LTV/CAC·payback과 월별 손익·현금은 startup-finance, 제품·서비스 제공원가와 gross margin 구축은 costing으로 보낸다. benchmark·attribution·incrementality를 생성하지 않고 계산 봉투만 인용한다.
---

# GTM Economics

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

원격 MCP의 `gtm-economics`로 채널 비용, 퍼널, 귀속 성과와 목표 역산을 계산한다.
전략 문서를 만드는 스킬이 아니라 사용자가 제공한 실제값·계획값의 분자, 분모, 기간과
측정 범위를 고정하는 계산 스킬이다.

## 라우팅 경계

| 요청 | 어디로 |
|---|---|
| 실제·계획 지출에서 채널 CAC·퍼널 비용·ROAS·ROMI·필요 리드/예산 계산 | `gtm-economics` (이 스킬) |
| GTM 채널 아이디어·시장 진입 전략·첫 100명 계획 | `startup-design` |
| 이미 확정된 CAC로 LTV/CAC·payback 계산 | `startup-finance` → `unit-economics` |
| 채널 예산·획득 고객을 월별 손익·현금으로 전개 | `startup-finance` → `financial-model` |
| 재료·서비스 시간·API·간접비에서 제공원가와 gross margin 구축 | `costing` |
| 가격 변화에 따른 수요·이익 | `pricing` |
| 자동 attribution·MMM·incrementality 추정 | 범위 밖. 검증된 외부 결과만 입력받는다 |

## 첫 호출 계약

1. `fintool_catalog {"tool":"gtm-economics"}`를 호출한다.
2. `input_contract.call_example`을 복사하고 사용자가 준 값만 바꾼다.
3. `fintool_run`의 `flags.spec`에 `gtm-economics/v1` 객체를 그대로 넣는다.

로컬 바이너리, 계산기, 채팅 산술로 결과를 대신 만들지 않는다. `--batch`가 필요하면 spec
안의 숫자 leaf 경로를 쓰고, 배열 원소는 index가 아니라 channel·cost·stage의 stable ID로
내려간다.

## 질문 순서

사용자가 이미 답한 항목은 다시 묻지 않는다. 다음 필수 block 하나씩 진행하며 한 번에 모든
질문을 던지지 않는다. 실제 입력이 없거나 모르면 0이나 시장 평균으로 채우지 않는다.

1. **결정**: 채널 비교, 예산 승인, 목표 역산, 실제 성과 리뷰 중 무엇을 결정하려는가.
2. **기간과 단위**: 비용 기간, cohort 기간, outcome cutoff, 통화, person·account·deal·customer 단위는 무엇인가.
3. **motion과 채널**: `paid_media`, `plg`, `sales_led`, `partnership`, `event` 중 무엇이며 stable channel ID는 무엇인가.
4. **비용 범위**: `media_only`, `campaign_fully_loaded`, `sales_marketing_fully_loaded`에 들어갈 비용과 acquisition·retention·COGS 분류는 무엇인가.
5. **퍼널 정의**: stage 순서, 진입 정의, count basis, dedupe key, 같은 entry cohort인지 확인한다.
6. **성과 귀속**: 신규 유료고객 정의, attributed customer/revenue, attribution method, lookback window, 포함·제외 채널을 확인한다.
7. **이익 연결**: gross margin이 실제로 있으면 gross-profit ROAS·ROMI와 downstream payback에 쓴다.
8. **incrementality gate**: 실험 또는 검증 모델의 method·reference·incremental outcome이 있는지 확인한다. 없으면 attributed 지표만 계산한다.
9. **목표 역산**: `target`이면 목표 고객·매출·pipeline 중 anchor 하나와 계획 stage rate·budget driver를 묻는다.
10. **검산과 인계**: reconciliation, unsupported evidence, provisional 기간을 먼저 보여주고 `unit-economics`·`financial-model` handoff가 필요한지 확인한다.

## Motion별 입력

없는 stage를 0으로 만들지 않는다. 실제로 존재하는 퍼널만 순서대로 넣는다.

| motion | 최소 퍼널·비용 | 추가로 확인할 정의 |
|---|---|---|
| `paid_media` | impression→click→conversion 또는 lead→win, media spend | platform count, conversion action, attribution window |
| `plg` | visitor/signup→activated→PQL 또는 paid, growth labor·tool | activation·PQL, free→paid, user/account dedupe |
| `sales_led` | lead→MQL→SQL→opportunity→win, sales labor·commission·CRM | person/account/deal mapping, sales cycle cutoff |
| `partnership` | referred account→opportunity→win, referral fee·MDF·partner labor | sourced/influenced, partner ID, commission basis |
| `event` | invite/registration→attendee→lead→opportunity→win, venue·sponsorship·staff | attendee dedupe, post-event window, sourced/influenced |

- `paid_media`: purchase가 아닌 platform conversion을 신규 유료고객으로 바꾸지 않는다.
- `plg`: signup user와 paying account를 같은 count로 쓰지 않는다. sales assist는 shared allocation 또는 별도 hybrid channel로 표현한다.
- `sales_led`: 조직의 MQL·SQL 정의와 loaded sales cost의 acquisition allocation을 받는다.
- `partnership`: influenced deal을 전액 sourced revenue로 세지 않고, 매출연동 fee의 acquisition/COGS 분류를 사용자에게 묻는다.
- `event`: registration·attendee·badge scan·lead를 같은 count로 쓰지 않는다. 직접 귀속이 없으면 attendee cost까지만 계산한다.

## Evidence 계약

| 입력 | evidence |
|---|---|
| 플랫폼 spend·impression·click·conversion | `source`: provider, dataset, as_of, conversion action |
| CRM stage·win·revenue | `source`: report, cohort cutoff, dedupe key |
| 급여·commission·tool·event invoice | `source`: GL, payroll, vendor export |
| acquisition allocation rate | `assumption`: 배부 근거와 승인 또는 time record |
| 계획 conversion rate·CPC·deal value | `assumption` 또는 historical run을 가리키는 `derived` |
| attributed revenue | `source`: attribution method·window·limitation |
| incremental customer·revenue | 실험·검증 결과를 가리키는 `derived`: method·reference 필수 |
| 외부 benchmark | 사용자가 채택했을 때만 `assumption`: URL·as_of·시장 적합성 경고 |

spec의 `evidence[].fields`는 `channels.<channel_id>.costs.<cost_id>.gross_amount` 같은 stable ID
경로를 쓴다. MCP 호출의 evidence는 catalog가 준 `fields`·`ref` 계약을 따른다. 근거가 없는
일반 입력은 `unsupported`로 남기되, incremental outcome과 target의 핵심 rate·driver는 evidence
없이 만들지 않는다.

benchmark, attribution rule, incrementality 결과를 생성하지 않는다. 사용자가 준 원장·보고서·
실험 결과를 변환할 뿐이며, 누락값을 업계 평균이나 암묵적 0으로 보충하지 않는다.

## 결과 scope

결과는 계산 봉투의 필드 이름과 metric metadata를 그대로 인용한다.

- **attributed / incremental**: `roas_attributed`와 `roas_incremental`,
  `romi_attributed_campaign_fully_loaded`와 `romi_incremental_campaign_fully_loaded`를
  각각 표시한다. attributed를 인과 효과라고 부르지 않는다.
- **paid / blended**: channel의 `paid_cac_media_only`·`paid_cac_campaign_fully_loaded`와
  company total의 `blended_cac_sales_marketing_fully_loaded`를 분리한다. 고객 분모와
  `new_customer_definition`을 함께 쓴다.
- **ROAS / ROMI**: ROAS는 `media_only` spend, ROMI는 `campaign_fully_loaded` investment의
  gross-profit basis다. `numerator`, `denominator`, `scope`, `measurement_basis`, `formula`를
  결과 옆에 명시한다.
- **기간**: cost period, cohort period, outcome cutoff, lookback window와
  `complete|provisional`을 함께 쓴다.

비율을 채팅에서 다시 평균내거나 반올림값으로 재계산하지 않는다. `null` metric은 0이나 무한대로
바꾸지 않고 봉투의 explanation과 warning을 전한다. 해시는 `calculation_hash` 전문을 인용한다.

## 검산과 handoff

먼저 `reconciliation.cost_difference`, `customer_credit_difference`, `revenue_difference`가 모두
0인지 확인한다. 차이가 있으면 결과를 의사결정값으로 쓰지 않는다. 그다음 unsupported evidence와
provisional warning을 보여준다.

- `unit_economics` handoff는 `blended|paid` CAC kind와
  `media_only|campaign_fully_loaded|sales_marketing_fully_loaded` scope를 보존한다.
- ARPA·gross margin·churn·period가 없으면 CAC만 partial로 넘기고 추정하지 않는다.
- `financial_model` handoff는 사용자가 준 월별 allocation 합이 source total과 일치할 때만 쓴다.
  기간 합계를 임의로 월별 균등 배분하지 않는다.
- 두 handoff의 `source_calculation_hash`와 derived evidence를 downstream 결과에 남긴다.

## 가드레일

- MQL·SQL·PQL·Opportunity 정의를 CRM 기본값으로 덮어쓰지 않는다.
- onboarding·customer success·platform fee를 CAC나 COGS로 자동 분류하지 않는다.
- channel win 합계를 deduped company customer로 간주하지 않는다.
- 최근 미성숙 cohort를 완전 기간처럼 비교하지 않는다.
- target의 conversion rate·CPC·deal value가 없으면 계산을 멈추고 다음 필수 질문을 한다.
