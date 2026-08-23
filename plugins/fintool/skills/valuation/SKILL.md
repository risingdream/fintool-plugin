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

할인율·영구성장률이 없으면 **가정해서 계산하지 않는다.** WACC 구성요소(무위험수익률·베타·ERP·세전 부채비용·세율·E/D 시장가치)와 영구성장률을 물어보고 멈춘다. FCF·순부채·주식 수만 있는 질문이 전형적이다. 사용자가 "그냥 10%로 해"라고 정하면 그 값을 `source: 사용자 지정 YYYY-MM-DD`로 적고 `dcf`를 돌린다.

`dcf` 호출 예(FCF 연 단위, 원):

```json
{"tool":"dcf","flags":{"fcf":"1000000000,1200000000,1400000000,1600000000,1800000000","wacc":0.1,"terminal-method":"growth","terminal-growth":0.02,"net-debt":2000000000,"shares":1000000}}
```

## 함정

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
