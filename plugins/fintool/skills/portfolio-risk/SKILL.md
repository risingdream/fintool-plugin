---
name: portfolio-risk
description: fintool MCP로 포트폴리오 최적화·위험·거래비용을 계산한다. 최적 비중, 공분산·연율 변동성, VaR, CVaR, 몬테카를로 고갈 확률, implementation shortfall·거래비용 분석(TCA)을 요청하면 이 스킬을 쓴다. 자산배분, 효율적 투자선, 리스크 예산, 손실 한도, 은퇴자금 고갈, 체결 슬리피지 같은 한국어 요청에 반응한다. 시계열 없이 연율 변동성·상관계수·공분산 행렬만 주어진 배분·손실 질문도 여기서 그대로 계산한다. Brinson 성과귀속·샤프·정보비율·TWRR·PE 펀드지표는 fund-performance, DCF·WACC는 valuation, 재무비율·부도확률은 financial-statements, 채권 가격·듀레이션은 fixed-income, 스타트업 IR은 startup-finance, 옵션 가격·그릭스·내재변동성·변동성 스마일은 option으로 보낸다. 숫자는 봉투만 인용한다.
---

# Portfolio Risk

원격 MCP: `fintool_catalog` → `fintool_run`.

범위: `vol` → `portfolio` → `var` / (선택) `montecarlo`. 거래비용은 `trade-cost`.  
범위 밖: 성과귀속·위험조정 성과지표·펀드지표(`fund-performance` — `attribution`·`perf`·`private-markets`), DCF·WACC(`valuation`), 재무비율·부도확률(`financial-statements`), 채권 프라이싱(`fixed-income`), IR HTML(`startup-finance`), 옵션 가격·그릭스·내재변동성(`option`).

## 흐름

```
시계열(returns/prices) 또는 모수(volatilities+correlation / covariance)
           → vol (시계열일 때만)
           → portfolio (목적함수, Σw=1)
           → var (손실 크기) 또는 montecarlo (경로·고갈)
```

가능하면 workflow `$ref`로 `annual_volatility_decimal`을 잇는다. `--seed`가 있는 호출은 시드를 고정한다.

## 스타트업 런웨이는 startup-finance로 보낸다

사업 매출·비용·채용·조달 계획에서 현금 고갈이나 런웨이를 묻는다면 **`startup-finance`**의 사업 재무모델 경계다. 은퇴자산·투자 포트폴리오처럼 자산 수익률 순서가 고갈을 좌우할 때만 이 스킬의 `montecarlo`를 쓴다.

- GBM 잔고 Monte Carlo는 자산 수익률 분포와 집계 현금흐름을 모델링할 뿐, 사업 매출 성장률·원가율·채용 불확실성을 직접 모델링하지 않는다. 스타트업의 기존 계획을 `annual-flow` 하나로 축약해 사업 확률처럼 해석하지 않는다.
- `mean-return=0`, `volatility=0`이면 모든 경로가 같아 **결정론적 스트레스**다. 고갈률을 확률 예측처럼 표현하지 않는다.
- 변동성·부트스트랩 표본·시나리오 확률처럼 경로를 가르는 입력이 있으면 무엇의 불확실성인지 밝히고 **분포 가정**으로 표시한다. 사업 드라이버 분포가 필요하면 `startup-finance`에서 `financial-model` 시나리오를 먼저 정의한다.

## 첫 계산 호출의 입력 형태

- `subcommand`는 여러 단계를 잇는 `workflow`에서만 쓴다. `portfolio`·`montecarlo`·`var`·`trade-cost`의 계산 호출에는 넣지 않는다.
- `portfolio --returns`의 다자산 시계열은 세미콜론 문자열이 아니라 JSON 객체다. 예: `"returns":{"주식":[0.01,-0.02],"채권":[0.003,0.004]}`.
- 시계열 `--returns`와 모수 입력 `--assets`·`--volatilities`·`--correlation`은 동시에 주지 않는다. `--assets`는 모수 입력에서만 쓴다.

## 성과 귀속은 이 스킬이 아니다

벤치마크 대비 초과수익 분해(`attribution`), 샤프·정보비율·TWRR/MWRR(`perf`), PE/VC 펀드지표(`private-markets`)는
**`fund-performance` 스킬**이 담당한다. 사용자가 "성과 귀속", "초과수익 분해", "샤프", "정보비율", "펀드 수익률"을 물으면
여기서 추측하지 말고 그 스킬의 절차서를 따른다. 이 스킬은 사전적(ex-ante) 최적화·위험과 실행비용만 다룬다.

## 옵션 IV는 이 스킬이 아니다

콜·풋 가격, 그릭스, 내재변동성, 변동성 스마일은 **`option` 스킬**이 담당한다.
`vol`은 수익률 시계열의 실현 변동성이지 옵션 IV가 아니다. 로컬 Python/SciPy로 역산하지 않는다.
사용자가 "콜옵션", "내재변동성", "변동성 스마일", "그릭스"를 물으면 이 문서의 예시로 계산하지 말고
`fintool_catalog {"tool":"option"}` 후 `fintool_run {"tool":"option"}`을 따른다.

## 모수만 주어졌을 때 — 시계열을 지어내지 않는다

"연 변동성 22%·15%·9%, 상관계수 0.4·0.1·0.25, 기대수익률 11%·7%·4%"처럼 **모수만** 있는 질문이 실무에서 가장 흔하다.
`portfolio`와 `var`는 이것을 **그대로 받는다.** Cholesky로 표본을 뽑아 CSV를 만들지 마라 — 표본공분산이 입력 모수와
달라 답이 매번 달라진다.

```json
{"tool":"portfolio","flags":{"objective":"max-sharpe","risk-free":0.03,
 "assets":["주식","채권","현금"],"mean-returns":[0.11,0.07,0.04],
 "volatilities":[0.22,0.15,0.09],"correlation":[[1,0.4,0.1],[0.4,1,0.25],[0.1,0.25,1]]}}
```

```json
{"tool":"var","flags":{"volatilities":[0.22,0.15],"correlation":[[1,0.4],[0.4,1]],
 "weights":[0.6,0.4],"confidence":0.99,"portfolio-value":1000000000}}
```

- `--volatilities` + `--correlation`(생략하면 무상관) 또는 `--covariance` 중 하나. **둘을 같이 주면 입력 오류다.**
- 모수는 전부 **연율 소수**다(22% = `0.22`). `--assets`를 생략하면 `asset1..N`이다.
- 시계열(`--returns`/`--prices`)과 **동시에 줄 수 없다.**
- `--mean-returns` 없이 낼 수 있는 해는 `min-variance`와 `risk-parity`뿐이다. 나머지는 μ를 거치므로 거절된다.
- `objective capm`은 벤치마크와의 회귀라 모수로 계산할 수 없다. 시계열을 요청한다.
- `var`는 모수 입력에서 **경험분포가 없어 `historical`을 못 낸다.** 방식을 고르지 않으면 `parametric`으로 계산하고 알린다.
  정규분포 가정이라 팻테일이 빠진다는 사실을 해설에 반드시 적는다.
- `var`의 모수는 연율이고 결과는 1기간이다. `--periods-per-year`(기본 252)로 환산하며 `--horizon`은 그 기간을 센다.
  일간 기준 `--horizon 10`이면 10일 VaR다.
- 결과의 `moments` 블록에 `determinant`·`min_eigenvalue`·`condition_number`가 들어온다. 조건수가 크다는 경고가 붙으면
  비중이 입력 모수의 작은 변화에도 흔들린다는 뜻이므로 해설에 인용한다.
- 상관행렬이 양의 정부호가 아니면 계산하지 않고 최소 고윳값·행렬식과 함께 거절한다. 사용자가 준 상관계수가 서로
  모순된다는 뜻이므로, 임의로 고치지 말고 어느 쌍이 문제인지 물어본다.

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
- **모수만 있어도 시계열을 지어내지 않는다.** `--volatilities`/`--correlation`/`--covariance`로 그대로 넘긴다(위 절차서 참고). 손으로 닫힌형을 푸는 것도 답이 아니다 — 그 값은 fintool 봉투가 아니다.
- 표본이 자산 수보다 짧으면 공분산이 불안정하다. 경고를 해설에 인용한다.
- **금액을 원 단위 정수로 바꿀 때 자릿수를 다시 센다.** "5,000억"은 `500000000000`(0이 11개)이다. 한 자리만 틀려도 엔진은 그대로 계산하고 결과는 10배 어긋난다. 헤지·포트폴리오 도구는 `notional_exposure`를 함께 내므로 원래 문제의 금액과 대조해 검산한다.
- `montecarlo`는 `summary: true`로 부른다. 연도별 분포 밴드(`path`)가 봉투의 70%를 넘고, 해설에는 `ruin_rate`·`terminal_balance` 분위수·`failure_analysis`만 있으면 된다. 연도별 밴드를 표로 그릴 때만 전체를 받는다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
