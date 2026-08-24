---
name: fund-performance
description: fintool MCP로 펀드 운용성과를 계산한다. Brinson 성과귀속(배분·선택·상호작용), 샤프·정보비율 등 위험조정 성과지표, TWRR·MWRR, PE/VC 펀드지표(IRR·TVPI·DPI·RVPI·PME), LP 분기 보고서 숫자를 요청하면 이 스킬을 쓴다. 성과귀속, 초과수익 분해, 벤치마크 대비 성과, 액티브 리턴, 정보비율, 펀드 수익률, 출자자 보고, 캐피털콜·분배, 워터폴·캐리, VC 펀드 성과, 바이아웃 IRR 같은 한국어 요청에 반응한다. 비중 최적화·VaR·거래비용은 portfolio-risk, DCF·WACC는 valuation, 스타트업 런웨이·캡테이블·IR HTML은 startup-finance로 보낸다. 숫자는 지어내지 않고 fintool 봉투만 인용한다.
---

# Fund Performance

도구는 **원격 MCP**다. `fintool_catalog` → `fintool_run`. 로컬 바이너리를 설치하지 않는다.

범위: `attribution`(성과귀속) · `perf`(위험조정 성과·TWRR/MWRR·GIPS) · `private-markets`(PE/VC 펀드지표·워터폴). PME는 `npv`·`irr`로 조립한다.
범위 밖: 최적 비중·VaR·거래비용(`portfolio-risk`), DCF·WACC(`valuation`), 런웨이·캡테이블·IR HTML(`startup-finance`).

## 원칙

1. **숫자를 만들지 않는다.** 계산은 전부 fintool이 하고, 해설은 봉투 필드만 인용한다. 봉투에 없는 값을 문장에 쓰지 않는다.
2. 사용자가 준 값과 내가 정한 가정을 구분해 적는다.
3. HTML을 직접 쓰지 않는다. 산출은 도구 호출과 해설이다.

## 흐름

```
1. 인터뷰   무엇을 보고하나 — 벤치마크 대비 초과수익? 위험조정 성과? 펀드 배수?
2. 정렬     기간·통화·주기·기준일을 맞춘다 (아래 sanity 규칙)
3. 계산     attribution / perf / private-markets 호출
4. 검산     합계 항등식·reconciled 플래그 확인
5. 해설     봉투 필드만 인용하고 관례(어느 방법을 썼는지)를 밝힌다
```

## 인터뷰 — 먼저 확정할 것

한 번에 묶어서 묻고, 답이 없으면 기본값을 정하고 그 사실을 답 앞머리에 굵게 적는다.

| 질문 | 없을 때 |
|---|---|
| 대상이 상장 포트폴리오인가 사모펀드인가 | 현금흐름 표(캐피털콜·분배)가 있으면 사모, 수익률 시계열이면 상장 |
| 수익률 주기 (일·주·월·분기) | 묻는다. **추측하지 않는다** — `--periods-per-year`가 결과를 통째로 바꾼다 |
| 벤치마크 시계열이 있는가 | 없으면 정보비율·베타·알파는 `null`로 나오고 오류가 아니다. 해설에 "벤치마크 미제공"이라 적는다 |
| gross인가 net인가 (운용보수 차감 전후) | gross로 두고 `--management-fee`로 net을 함께 낸다 |
| 사모펀드면 약정액·관리보수율·hurdle·캐리율 | 시장 관행 2/20, hurdle 8%, full catch-up. 가정임을 밝힌다 |

## 단위·통화·기간 sanity 규칙

계산 전에 매번 확인한다. 엔진은 단위를 검증하지 못하고 그대로 계산한다.

- **모든 비율은 소수다.** 8.4%는 `0.084`. `8.4`를 넣으면 840%로 계산된다. 수익률·hurdle·캐리율·할인율 전부 같다.
- **`--periods-per-year`를 항상 명시한다.** 기본값이 252(일간)라 월간 12개 수익률을 그대로 넣으면 `cagr`이 18.13(=1813%)으로 나온다. 일간 거래일 252, 달력일 365, 주간 52, 월간 12, 분기 4.
- **연율인지 기간율인지 구분한다.** `--risk-free`·`--mar`는 연율 소수로 주고, 엔진이 기간율로 환산한다(`data.annualization`에 환산 결과가 나온다). 반대로 `private-markets --mode fund`의 `irr`은 **기간 IRR**이다 — 아래 함정 참고.
- **통화·배율을 한 표에서 통일한다.** `--gross-cashflows`·`--residual-value`·`--committed-capital`은 같은 통화, 같은 배율(전부 원 또는 전부 백만원)이어야 한다. 섞으면 TVPI가 조용히 틀린다.
- **원 단위 정수는 자릿수를 다시 센다.** "5,000억"은 `500000000000`(0이 11개)이다. 한 자리 어긋나면 결과가 10배 틀린다.
- **기준일을 맞춘다.** `perf --returns`와 `--benchmark`는 길이·주기·시작일이 같아야 한다. `attribution`의 세그먼트는 기간마다 같은 이름 집합을 쓴다.
- **비중은 기간마다 합이 1이다.** 현금 비중을 빠뜨리면 `attribution`이 거절한다.
- **날짜가 불규칙하면 기간 인덱스가 아니라 `--dates`를 쓴다.** `irr`·`npv`는 `--dates`로 XIRR·XNPV가 된다. `private-markets`는 `--dates`를 받지 않는다.

## 성과귀속 — `attribution`

### Brinson (주식·자산배분)

벤치마크 대비 초과수익을 배분(allocation)·선택(selection)·상호작용(interaction)으로 쪼갠다.

```json
{"tool":"attribution","flags":{"model":"brinson","brinson-method":"brinson-fachler","linking":"carino","input":{
 "periods":[
  {"name":"2026Q1","segments":[
    {"name":"IT","portfolio_weight":0.32,"benchmark_weight":0.25,"portfolio_return":0.081,"benchmark_return":0.064},
    {"name":"금융","portfolio_weight":0.18,"benchmark_weight":0.22,"portfolio_return":0.021,"benchmark_return":0.035},
    {"name":"기타","portfolio_weight":0.50,"benchmark_weight":0.53,"portfolio_return":0.030,"benchmark_return":0.028}]},
  {"name":"2026Q2","segments":[
    {"name":"IT","portfolio_weight":0.35,"benchmark_weight":0.26,"portfolio_return":0.042,"benchmark_return":0.038},
    {"name":"금융","portfolio_weight":0.15,"benchmark_weight":0.21,"portfolio_return":0.011,"benchmark_return":0.019},
    {"name":"기타","portfolio_weight":0.50,"benchmark_weight":0.53,"portfolio_return":0.025,"benchmark_return":0.022}]}]}}}
```

- 필드는 snake_case다. `sectors`·`holdings`·`groups`·`portfolioWeight`는 받지 않는다.
- `--model`은 **필수**다. 빼면 거절당한다.
- `--brinson-method`: `brinson-fachler`는 배분효과를 벤치마크 전체수익률 초과분 기준(`(wP−wB)(rB,i−RB)`)으로, `brinson-hood-beebower`는 `(wP−wB)·rB,i`로 잡는다. **어느 쪽을 썼는지 해설에 밝힌다.** 총 액티브는 같고 배분·선택 사이 배분만 달라진다.
- 다기간이면 `periods`를 여러 개 넣고 `--linking carino`로 잇는다. 기간 액티브의 단순 합은 기하 누적과 다르고, Carino 계수가 그 차이를 잔차 없이 배분한다. 봉투의 `data.periods[].linking_factors`가 그 계수다.
- 인용: `data.summary.{portfolio_return, benchmark_return, active_return}`, `data.summary.segments[].{allocation, selection, interaction, active}`, `data.summary.totals`. 기간별은 `data.periods[]`.
- **검산**: `totals.allocation + totals.selection + totals.interaction = totals.active`, 그리고 `portfolio_return − benchmark_return = active_return`. 해설에 한 줄로 넣는다.
- 해석은 세 효과를 구분한다. 배분은 섹터 베팅, 선택은 섹터 안 종목 선정, 상호작용은 둘의 곱(과대비중 섹터에서 종목까지 좋았는가)이다.

### 고정수익 (듀레이션 기반)

`--model fixed-income`은 **완전히 다른 계약**이다. `periods`가 아니라 `securities`를 받는다.

```json
{"tool":"attribution","flags":{"model":"fixed-income","input":{
 "securities":[
  {"name":"국고 3년","weight":0.45,"total_return":0.0112,"income_return":0.0072,"modified_duration":2.8,"curve_change":-0.0012,"spread_change":0.0002},
  {"name":"회사채 AA- 5년","weight":0.35,"total_return":0.0141,"income_return":0.0104,"modified_duration":4.4,"curve_change":-0.0012,"spread_change":-0.0005},
  {"name":"통안채 1년","weight":0.20,"total_return":0.0081,"income_return":0.0079,"modified_duration":0.9,"curve_change":-0.0008,"spread_change":0.0001}]}}}
```

- `curve_change`·`spread_change`는 **수익률 변화폭을 소수로** 준다. −12bp는 `-0.0012`다.
- 인용: `data.summary.{total_return_contribution, income_effect, curve_effect, spread_effect, residual_effect}`, 종목별은 `data.securities[]`.
- 1차 수정듀레이션 근사다(`method: first-order-modified-duration`). 컨벡시티를 안 잡으므로 금리가 크게 움직인 분기에는 `residual_effect`가 커진다. **잔차 크기를 반드시 해설에 인용하고, 크면 근사 한계라고 적는다.**

## 위험조정 성과 — `perf`

```json
{"tool":"perf","flags":{
 "returns":"0.021,-0.008,0.034,0.012,-0.019,0.027,0.005,0.031,-0.011,0.018,0.009,0.024",
 "benchmark":"0.018,-0.012,0.029,0.010,-0.022,0.024,0.002,0.028,-0.015,0.016,0.006,0.020",
 "periods-per-year":12,"risk-free":0.03,"management-fee":0.01}}
```

시계열이 길면 `"returns":"@fund.csv"`로 파일을 준다.

- 인용 위치를 헷갈리지 않는다:
  - 수익률: `data.returns.{cumulative_return, cagr, hit_rate, best_period, worst_period}`
  - 위험: `data.risk.{volatility_annual, downside_deviation_annual, var_loss, cvar_loss, skewness}`
  - 낙폭: `data.drawdown.{max_drawdown_loss, drawdown_periods, recovered}`
  - 위험조정: `data.risk_adjusted.{sharpe, sortino, calmar, omega}`
  - **벤치마크 대비**: `data.benchmark.{beta, alpha_annual, tracking_error_annual, information_ratio, treynor, up_capture, down_capture, r_squared}`
  - 보수: `data.cfa.fees.{gross_annualized_return, net_annualized_return}`
  - 초과수익 품질: `data.cfa.risk_adjusted.{appraisal_ratio, m2_return_annual, residual_risk_annual}`
- **정보비율은 `data.risk_adjusted`가 아니라 `data.benchmark.information_ratio`에 있다.** 벤치마크를 안 주면 이 블록 전체가 `null`이고 그건 오류가 아니다.
- `sharpe`는 초과수익 평균/표준편차를 연율화한 값, `information_ratio`는 액티브 수익/추적오차다. **둘을 섞어 부르지 않는다.**
- 표본이 짧으면 샤프·정보비율이 과장된다. 12개 관측치의 정보비율 13은 통계가 아니라 표본 부족이다. **관측치 수(`data.periods`)를 항상 함께 인용하고, 24개 미만이면 한계로 적는다.**
- `sortino`의 하방편차는 분모를 전체 표본 수 n으로 나눈다(empyrical·PerformanceAnalytics 관행). 다른 구현과 값이 다를 수 있다.
- 손실은 모두 **양수가 손실**이다. `max_drawdown_loss` 0.35 = 35% 하락.

### TWRR / MWRR — 외부 현금흐름이 있는 계좌

```json
{"tool":"perf","flags":{"returns":"@fund.csv","periods-per-year":12,
 "initial-value":100000000,"cashflows":"0,20000000,0,-10000000","cashflow-timing":"end"}}
```

- **부호가 `private-markets`와 반대다.** 여기서는 **양수 = 투자자 납입(포트폴리오 유입)**, 음수 = 인출. `private-markets --gross-cashflows`는 LP 관점이라 납입이 음수다. 같은 펀드 데이터를 두 도구에 넣을 때 부호를 뒤집어야 한다.
- 인용: `data.cfa.performance_returns.{twrr, mwrr_period, mwrr_annual, mwrr_unique, terminal_value}`.
- `mwrr_unique`가 `false`면 부호가 여러 번 바뀌어 해가 여럿이다. 그 값을 단독 결론으로 쓰지 않는다.
- **어느 쪽을 보고할지 정한다.** 운용사 실력 평가는 TWRR(현금흐름 타이밍 효과 제거), 출자자 실현 수익은 MWRR이다. 둘이 벌어지면 그 차이가 곧 납입 타이밍 효과이므로 해설에 그렇게 쓴다.
- GIPS 복합체는 `--composite-values`에 구성 포트폴리오별 기초 시장가치를 주고 `data.cfa.composite`를 인용한다.

## PE/VC 펀드지표 — `private-markets`

### 펀드 성과·워터폴 (`--mode fund`)

```json
{"tool":"private-markets","flags":{"mode":"fund",
 "gross-cashflows":"-30000000000,-25000000000,-15000000000,12000000000,28000000000",
 "residual-value":55000000000,"committed-capital":100000000000,
 "management-fee-rate":0.02,"management-fee-base":"committed",
 "hurdle-rate":0.08,"carried-interest-rate":0.20,"catch-up-rate":1}}
```

- `gross-cashflows`는 **LP 관점 gross 흐름**이다. 캐피털콜은 음수, 분배는 양수. 보수·캐리 차감 전 값을 준다.
- `residual-value`는 보고시점 gross NAV다. 엔진이 같은 시점 청산으로 가정해 RVPI와 IRR에 반영한다.
- 인용: `data.gross.{paid_in, distributions, residual_value, dpi, rvpi, tvpi, irr}`와 같은 모양의 `data.net`. 워터폴은 `data.waterfall[]`, 보수·캐리 합계는 `data.fees.{total_management_fees, realized_carry, accrued_carry, total_carry}`.
- **LP 보고서에는 `data.net`을 쓴다.** `data.gross`는 GP 운용 실력, `data.net`이 출자자가 실제로 받는 값이다. 어느 쪽인지 표에 반드시 라벨을 단다.
- **검산 두 가지를 매번 한다.** `tvpi = dpi + rvpi`, 그리고 `data.reconciliation.reconciled == true`. `reconciled`가 false면 결과를 인용하지 않고 입력을 다시 본다.
- `data.assumptions`를 그대로 해설에 옮긴다: `european-whole-fund` 워터폴, `annual-compounded` hurdle, 보수는 최종 기간을 뺀 각 기간 시작에 부과, **recycling 미지원·clawback 미모델링**. 딜별(American) 워터폴이나 리사이클링이 있는 펀드면 이 결과가 실제와 다르다고 명시한다.
- `net.paid_in`은 gross 납입에 관리보수를 더한 값이라 `gross.paid_in`보다 크다. 두 배수의 분모가 다르다는 뜻이므로 나란히 놓을 때 각주를 단다.

### ⚠️ fund의 `irr`은 기간 IRR이다

`private-markets`는 `--dates`를 받지 않는다. `gross-cashflows`를 기간 인덱스 0,1,2…로만 읽으므로 **나온 `irr`은 한 기간당 수익률**이다.

- 연 단위 흐름이면 그대로 연율이다.
- 분기 흐름이면 연율은 `(1+irr)^4 − 1`이다. **분기 IRR을 연 IRR이라고 쓰지 않는다.**
- 캐피털콜·분배 날짜가 불규칙한 실제 펀드라면 연율 IRR은 `irr --dates`로 따로 낸다:

```json
{"tool":"irr","flags":{
 "cashflows":"-30000000000,-25000000000,-15000000000,12000000000,28000000000,55000000000",
 "dates":"2021-03-31,2021-09-30,2022-06-30,2023-06-30,2024-06-30,2025-12-31"}}
```

마지막 흐름에 잔존 NAV를 양수로 넣는다(같은 날 청산 가정). 인용은 `data.rate`와 `data.unique`. `unique`가 false면 부호가 여러 번 바뀐 것이므로 배수(TVPI)를 함께 보고한다.

### PME — 전용 도구가 없다, `npv`로 조립한다

fintool에 PME 도구는 없다. **PME 숫자를 지어내지 않는다.** 대신 두 경로 중 하나를 쓰고 어느 쪽인지 밝힌다.

**(1) KS-PME** — 같은 `--dates`로 `npv`를 두 번 부른다. 할인율은 같은 기간 지수의 연율 수익률이다.

```json
{"tool":"npv","flags":{"rate":0.084,
 "cashflows":"30000000000,25000000000,15000000000,0,0,0",
 "dates":"2021-03-31,2021-09-30,2022-06-30,2023-06-30,2024-06-30,2025-12-31"}}
```

```json
{"tool":"npv","flags":{"rate":0.084,
 "cashflows":"0,0,0,12000000000,28000000000,55000000000",
 "dates":"2021-03-31,2021-09-30,2022-06-30,2023-06-30,2024-06-30,2025-12-31"}}
```

KS-PME = (분배+NAV의 PV) / (납입의 PV). 두 `data.value`를 나눈 값 하나만 직접 계산하고, **두 PV를 모두 표에 적어 검산이 되게 한다.** 1보다 크면 지수 대비 초과, 작으면 미달이다.

- **두 호출의 `--dates`가 완전히 같아야 한다.** XNPV는 첫 날짜를 기준일로 삼아 할인하지 않으므로, 날짜 벡터가 다르면 두 PV의 기준일이 어긋나 비율이 무의미해진다. 그래서 각 호출에서 해당하지 않는 흐름을 `0`으로 채운다.
- 할인율에 쓸 지수 수익률은 지어내지 말고 `perf --prices <지수 시계열>`의 `data.returns.cagr`을 쓴다.

**(2) IRR 스프레드** — 펀드 XIRR(`irr --dates`의 `data.rate`)에서 같은 기간 지수 CAGR(`perf`의 `data.returns.cagr`)을 뺀다. 해석이 직관적이지만 KS-PME와 달리 납입 규모 가중이 아니다. **어느 정의를 썼는지 반드시 적는다.**

### 그 밖의 모드

| 모드 | 쓰는 곳 | 핵심 플래그 | 인용 |
|---|---|---|---|
| `vc` | VC method 역산 — 목표 IRR을 맞추려면 지분 몇 %가 필요한가 | `investment`, `exit-metric`, `exit-multiple`, `net-debt-at-exit`, `years`, `target-return`, `future-dilution` | `required_ownership_post_money`, `post_money_value`, `pre_money_value`, `target_moic` |
| `growth` | 그로스 지분 가치 | `current-metric`, `entry-multiple`, `exit-multiple`, `metric-growth`, `net-debt`, `years`, `required-return` | `entry_equity_value`, `exit_equity_value`, `present_equity_value` |
| `buyout` | LBO 에쿼티 브리지 | `entry-metric`, `entry-multiple`, `entry-debt`, `exit-multiple`, `exit-debt`, `metric-growth`, `years` | `entry_equity_value`, `exit_equity_value`, `debt_paydown`, `moic`, `irr` |
| `debt` | 사모대출 가치·커버리지 | `principal`, `coupon-rate`, `years`, `required-return`, `collateral-value`, `ebitda`, `interest-expense`, `cfads`, `debt-service` | `fair_value`, `dscr`, `interest_coverage`, `ltv`, `current_yield` |
| `asset` | 실물자산 terminal cap | `asset-class`, `cashflows`, `discount-rate`, `terminal-noi`, `exit-cap-rate` | `npv`, `irr`, `terminal_value` |

- `growth`는 `--investment`를 **거절한다**. 투자금이 아니라 지표·배수로 지분가치를 낸다.
- `buyout`·`vc`의 `moic`·`irr`은 `years`를 연 단위로 준 경우에만 연율이다.
- `asset`의 `cashflows[0]`은 시점 0의 음수 투자금이다. terminal value는 마지막 기간에 더해진다.

## LP 분기 보고서 흐름

분기 리포트를 요청받으면 이 순서로 낸다. 각 줄의 숫자는 전부 봉투에서 온다.

```
1. 펀드 요약   private-markets --mode fund → data.net.{paid_in, distributions, residual_value}
2. 배수·수익률 data.net.{dpi, rvpi, tvpi} + irr --dates → data.rate (연율)
3. 벤치마크    KS-PME (npv 2회) 또는 IRR 스프레드
4. 보수·캐리   data.fees.{total_management_fees, realized_carry, accrued_carry}
5. 워터폴      data.waterfall[] 마지막 행의 lp_distribution / gp_carry
6. 검산        tvpi = dpi + rvpi, data.reconciliation.reconciled
7. 각주        data.assumptions 전체 + 미실현 NAV 평가 근거는 GP 제공값이라는 사실
```

- **DPI는 실현, RVPI는 미실현이다.** TVPI만 앞세우고 구성을 숨기지 않는다. TVPI 1.6이 DPI 0.2 + RVPI 1.4면 아직 회수된 게 거의 없다는 뜻이고, 보고서 본문에 그렇게 쓴다.
- 잔존가치는 GP가 평가한 값이지 시장가가 아니다. 매 분기 각주로 남긴다.
- 상장 파트가 함께 있는 혼합 포트폴리오면 상장 쪽은 `perf`, 사모 쪽은 `private-markets`로 각각 내고 **합산 수익률을 임의로 만들지 않는다.** 두 지표는 정의(TWRR vs 자금가중 IRR)가 달라 단순 가중평균이 성립하지 않는다.

## 카탈로그 호출 줄이기

`fintool_catalog`는 스펙 조회일 뿐 계산이 아니다. 계측에서 전체 도구 호출의 31%가 카탈로그였다.

- 이 문서에 **호출 예시가 있는 도구는 카탈로그를 부르지 않는다.** 예시를 그대로 쓰고 숫자만 바꾼다.
- 예시가 없는 도구만 `fintool_catalog {"tool":"<이름>"}`으로 그 도구 하나를 받는다. 인자 없는 전체 목록은 어떤 도구가 있는지 모를 때만 쓴다.
- 호출이 실패하면 오류 봉투가 유효 플래그 목록과 `input_contract`를 함께 준다. 그것을 보고 고치면 되고 카탈로그를 따로 부를 필요가 없다.

## 함정

- **`attribution --model`을 빠뜨리지 않는다.** 기본값이 없다.
- **`perf --periods-per-year`를 빠뜨리지 않는다.** 기본 252가 월간 데이터에 적용되면 CAGR이 1800%대로 나온다.
- **부호 규약이 도구마다 반대다.** `perf --cashflows`는 납입이 양수, `private-markets --gross-cashflows`는 납입이 음수.
- **`private-markets --mode fund`의 `irr`은 기간 IRR이다.** 분기 흐름이면 연율화하거나 `irr --dates`를 쓴다.
- **비중 합이 1이 아니면 `attribution`이 거절한다.** 현금 비중을 세그먼트로 넣는다.
- **`information_ratio`는 `data.benchmark` 아래에 있다.** 벤치마크 없으면 `null`이고 오류가 아니다.
- **PME 도구는 없다.** `npv`·`irr`로 조립하고 정의를 밝힌다. 값을 지어내지 않는다.
- **`net`과 `gross`를 섞지 않는다.** 분모(`paid_in`)가 다르다.
- 표본이 짧으면 샤프·정보비율·PME 전부 신뢰구간이 넓다. 관측치 수를 함께 낸다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
