---
name: portfolio-risk
description: fintool MCP로 포트폴리오 최적화·VaR·성과지표·성과 귀속·거래비용을 계산한다. 최적 비중, 샤프, VaR, CVaR, 고갈 확률, Brinson 성과 귀속(배분·선택 효과), implementation shortfall·거래비용 분석을 요청하면 이 스킬을 쓴다. 스타트업 IR은 startup-finance, DCF는 valuation으로 보낸다. 숫자는 봉투만 인용한다.
---

# Portfolio Risk

원격 MCP: `fintool_catalog` → `fintool_run`.

범위: `vol` → `portfolio` → `var` / `perf` / (선택) `montecarlo`. 성과 귀속은 `attribution`, 거래비용은 `trade-cost`.  
범위 밖: IR HTML, DCF, 채권 프라이싱.

## 흐름

```
시계열 입력 → vol 또는 그대로 returns
           → portfolio (목적함수, Σw=1)
           → var (손실 크기) 또는 perf
```

가능하면 workflow `$ref`로 `annual_volatility_decimal`을 잇는다. `--seed`가 있는 호출은 시드를 고정한다.

## 성과 귀속 — `attribution`

벤치마크 대비 초과수익을 배분(allocation)·선택(selection)·상호작용(interaction)으로 쪼갠다.
`fintool_catalog {"tool":"attribution"}`의 `input_contract`에 그대로 실행되는 예시가 있으니 필드명을 추측하지 않는다.

```json
{"tool":"attribution","flags":{"model":"brinson","brinson-method":"brinson-fachler","linking":"carino","input":{
 "periods":[{"name":"2026Q1","segments":[
   {"name":"IT","portfolio_weight":0.32,"benchmark_weight":0.25,"portfolio_return":0.081,"benchmark_return":0.064},
   {"name":"금융","portfolio_weight":0.18,"benchmark_weight":0.22,"portfolio_return":0.021,"benchmark_return":0.035},
   {"name":"기타","portfolio_weight":0.50,"benchmark_weight":0.53,"portfolio_return":0.030,"benchmark_return":0.028}]}]}}}
```

- 필드는 snake_case다. `sectors`·`holdings`·`groups`·`portfolioWeight`는 받지 않는다.
- **기간마다 `portfolio_weight` 합과 `benchmark_weight` 합이 각각 1**이어야 한다. 현금 비중을 빠뜨리면 여기서 걸린다.
- `--brinson-method`: `brinson-fachler`는 배분효과를 벤치마크 전체수익률 초과분 기준(`(wP−wB)(rB,i−RB)`)으로, `brinson-hood-beebower`는 `(wP−wB)·rB,i`로 잡는다. **어느 쪽을 썼는지 해설에 밝힌다.**
- 다기간이면 `periods`를 여러 개 넣고 `--linking carino`로 연결한다. 기간 액티브의 단순 합은 기하 누적과 다르며, Carino 계수가 그 차이를 잔차 없이 배분한다.
- 인용: `data.summary.{portfolio_return, benchmark_return, active_return}`, `data.summary.segments[].{allocation, selection, interaction, active}`, `data.summary.totals`. 기간별은 `data.periods[]`.
- 해석은 세 효과를 구분해서 쓴다. 배분은 섹터 베팅, 선택은 종목, 상호작용은 둘의 곱이다.

## 거래비용 — `trade-cost`

```json
{"tool":"trade-cost","flags":{"solve":"shortfall","side":"buy","order-quantity":500000,
 "decision-price":47200,"arrival-price":47450,
 "execution-prices":[47510,47680],"execution-quantities":[180000,150000],
 "close-price":48100,"explicit-costs":3800000}}
```

- `--solve`: `spread`(quoted/effective) · `vwap` · `shortfall`(4분해).
- shortfall 인용: `data.shortfall.{delay_cost, trading_cost, opportunity_cost, explicit_cost, implementation_shortfall}`. 각각 `.amount`와 `.basis_points`를 함께 낸다. `identity_residual`이 0인지 확인하고 해설에 검산으로 쓴다.
- 미체결분이 있으면 기회비용이 잡힌다. **지연·기회비용의 귀속 관례가 둘로 갈린다**(미체결분의 지연 손실을 지연에 넣느냐 기회에 넣느냐). 합계는 같으니 어느 관례인지 밝히고 쓴다.
- 매수는 `side buy`, 매도는 `sell`. 부호는 비용이 양수다.

## 카탈로그 호출 줄이기

`fintool_catalog`는 스펙 조회일 뿐 계산이 아니다. 계측에서 전체 도구 호출의 31%가 카탈로그였다.

- 이 문서에 **호출 예시가 있는 도구는 카탈로그를 부르지 않는다.** 예시를 그대로 쓰고 숫자만 바꾼다.
- 예시가 없는 도구만 `fintool_catalog {"tool":"<이름>"}`으로 그 도구 하나를 받는다. 인자 없는 전체 목록은 어떤 도구가 있는지 모를 때만 쓴다.
- 호출이 실패하면 오류 봉투가 유효 플래그 목록과 `input_contract`를 함께 준다. 그것을 보고 고치면 되고 카탈로그를 따로 부를 필요가 없다.

## 함정

- `portfolio`는 **공매도 금지를 지원하지 않는다.** `allows_short`는 항상 true. 음수 비중 = 공매도.
- `var_loss` / `cvar_loss`는 **양수가 손실**. 음수면 그 신뢰수준에서도 이익.
- `--returns`와 `--prices`를 섞지 않는다.
- 표본이 자산 수보다 짧으면 공분산이 불안정하다. 경고를 해설에 인용한다.
- **금액을 원 단위 정수로 바꿀 때 자릿수를 다시 센다.** "5,000억"은 `500000000000`(0이 11개)이다. 한 자리만 틀려도 엔진은 그대로 계산하고 결과는 10배 어긋난다. 헤지·포트폴리오 도구는 `notional_exposure`를 함께 내므로 원래 문제의 금액과 대조해 검산한다.
- `montecarlo`는 `summary: true`로 부른다. 연도별 분포 밴드(`path`)가 봉투의 70%를 넘고, 해설에는 `ruin_rate`·`terminal_balance` 분위수·`failure_analysis`만 있으면 된다. 연도별 밴드를 표로 그릴 때만 전체를 받는다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
