---
name: valuation
description: fintool MCP로 WACC·DCF·배수 가치평가를 한다. 내재가치, 기업가치, 주당가치, 할인율, 삼성전자형 DCF를 요청하면 이 스킬을 쓴다. 스타트업 런웨이·캡테이블·IR HTML은 startup-finance로 보낸다. 숫자는 fintool 봉투만 인용한다.
---

# Valuation

도구는 **원격 MCP**다. `fintool_catalog` → `fintool_run`. 로컬 바이너리를 설치하지 않는다.

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
