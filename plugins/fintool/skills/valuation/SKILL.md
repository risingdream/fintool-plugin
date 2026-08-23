---
name: valuation
description: fintool MCP로 WACC·DCF·배수 가치평가를 한다. 내재가치, 기업가치, 주당가치, 할인율, 삼성전자형 DCF를 요청하면 이 스킬을 쓴다. 스타트업 런웨이·캡테이블·IR HTML은 startup-finance로 보낸다. 숫자는 fintool 봉투만 인용한다.
---

# Valuation

도구는 **원격 MCP**다. `fintool_catalog` → `fintool_run`. 로컬 바이너리를 설치하지 않는다.
`fintool_catalog`는 인자 없이 부르면 목록만 준다. 플래그는 `{"tool": "wacc"}`처럼 도구를 지정해 받는다.

범위: 가정 → `wacc` → `dcf` → (선택) `equity`.  
범위 밖: 런웨이·BP·캡테이블·IR HTML (`startup-finance`).

## 원칙

1. LLM은 숫자를 만들지 않는다. 봉투만 인용한다.
2. 가정에 출처를 붙인다.
3. HTML을 직접 쓰지 않는다. 산출은 workflow JSON과 해설.

## 흐름

```
1. 가정     무위험·베타·ERP·세전부채비용·세율·E/D 시장가치, FCFF 경로
2. wacc     cost_of_equity, after_tax_cost_of_debt, wacc
3. dcf      --fcf 는 비레버드 FCFF. --wacc 는 2단계 결과
4. 선택     equity 배수 비교. DCF를 대체하지 않는다
5. 연결     workflow v2 명세로 wacc → dcf 를 typed $ref 로 고정
```

가능하면 `workflow validate` 후 `workflow run`을 쓴다. 예: `docs/examples/workflow-wacc-dcf.json`.

## 입력이 빠졌을 때

할인율·영구성장률이 없어도 **멈추지 않는다.** 가정을 세우고 계산하되, 가정을 숨기지 않는다.

1. 빠진 값마다 기본값과 근거를 정한다. WACC는 업종·시장 기준 범위(예: 한국 비상장 중소기업 10~12%, 상장 대형주 7~9%), g는 장기 물가+실질성장 기준 1~3%. 근거를 한 줄로 적는다.
2. `assumptions[].source`에 `기본값 제안 YYYY-MM-DD (사용자 미제공)`으로 남긴다. 사용자가 준 값은 `사용자 입력 YYYY-MM-DD`.
3. base를 `dcf`로 계산하고, **가정한 변수는 반드시 민감도 격자**(`wacc-range`, `terminal-growth-range`)로 같이 돌린다. 한 점 추정만 주지 않는다.
4. 답 맨 앞에 "**가정: WACC x%, g y% (제가 정한 값, 근거 …)**"를 굵게 쓰고, 끝에 "값을 주시면 다시 계산"을 덧붙인다.
5. 가정 없이는 결과 범위가 너무 넓은 경우(예: FCF도 없음)에만 질문으로 멈춘다.

`dcf` 호출 예(FCF 연 단위, 원):

```json
{"tool":"dcf","flags":{"fcf":"1000000000,1200000000,1400000000,1600000000,1800000000","wacc":0.1,"terminal-method":"growth","terminal-growth":0.02,"net-debt":2000000000,"shares":1000000,"wacc-range":"0.08,0.12","terminal-growth-range":"0.01,0.03"}}
```

## 함정
- **금액을 원 단위 정수로 바꿀 때 자릿수를 다시 센다.** "5,000억"은 `500000000000`(0이 11개)이다. 엔진은 단위를 검증하지 못하고 그대로 계산한다.


- **`business-plan.free_cash_flow`를 `dcf --fcf`에 넣지 않는다.** BP는 이자 차감 후 레버드 FCF다. dcf는 비레버드 FCFF다.
- `equity-value`·`debt-value`는 장부가가 아니라 **시장가치**.
- `--fcf` 첫 값은 기본이 1년차 말. 연중 평가면 `--valuation-date`와 잔여 stub만.
- `terminal-growth`는 wacc보다 작아야 한다.
- 국가위험·규모 프리미엄은 `wacc` 범위 밖. 넣지 말고 해설에 한계로 적는다.

## 호출

플래그는 kebab-case. `spec`은 문자열. 비율은 연율 소수.

```
wacc: risk-free-rate, beta, market-risk-premium, cost-of-debt, tax-rate, equity-value, debt-value
dcf:  fcf (쉼표 또는 @파일), wacc, terminal-growth 또는 exit multiple
```

결과는 `data.wacc`, `data.enterprise_value`, `data.equity_value`만 인용한다.
