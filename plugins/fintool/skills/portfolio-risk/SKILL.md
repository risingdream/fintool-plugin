---
name: portfolio-risk
description: fintool MCP로 포트폴리오 최적화·VaR·성과지표를 계산한다. 최적 비중, 샤프, VaR, CVaR, 고갈 확률을 요청하면 이 스킬을 쓴다. 스타트업 IR은 startup-finance, DCF는 valuation으로 보낸다. 숫자는 봉투만 인용한다.
---

# Portfolio Risk

원격 MCP: `fintool_catalog` → `fintool_run`.

범위: `vol` → `portfolio` → `var` / `perf` / (선택) `montecarlo`.  
범위 밖: IR HTML, DCF, 채권 프라이싱.

## 흐름

```
시계열 입력 → vol 또는 그대로 returns
           → portfolio (목적함수, Σw=1)
           → var (손실 크기) 또는 perf
```

가능하면 workflow `$ref`로 `annual_volatility_decimal`을 잇는다. `--seed`가 있는 호출은 시드를 고정한다.

## 함정

- `portfolio`는 **공매도 금지를 지원하지 않는다.** `allows_short`는 항상 true. 음수 비중 = 공매도.
- `var_loss` / `cvar_loss`는 **양수가 손실**. 음수면 그 신뢰수준에서도 이익.
- `--returns`와 `--prices`를 섞지 않는다.
- 표본이 자산 수보다 짧으면 공분산이 불안정하다. 경고를 해설에 인용한다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
