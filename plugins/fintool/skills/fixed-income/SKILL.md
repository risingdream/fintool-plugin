---
name: fixed-income
description: fintool MCP로 채권 가격·수익률·듀레이션·커브·전환사채를 계산한다. 이표채, 할인채, 국채, Z-spread, 커브 부트스트랩, 전환사채(CB)·전환옵션 가치를 요청하면 이 스킬을 쓴다. 스타트업 IR은 startup-finance, DCF는 valuation으로 보낸다. 숫자는 봉투만 인용한다.
---

# Fixed Income

원격 MCP: `fintool_catalog` → `fintool_run`.

범위: `coupon` / daycount basis → `bond` / `discount` → (선택) `curve`. 전환사채는 `convertible`.  
범위 밖: 주식 DCF, 스타트업 런웨이.

## 흐름

```
1. 일수 계산 기준(--basis 0~4)을 정한다
2. 이표 스케줄이 필요하면 coupon
3. 가격·수익률·듀레이션은 bond. 할인채·T-bill은 discount
4. 곡선·스팟·포워드·Z-spread는 curve
```

## 전환사채(CB)

`convertible` 하나로 평가한다. Tsiveriotis-Fernandes 이항 트리라 발행사 신용스프레드가 전환권에 반영되고, 발행사 콜·투자자 풋·만기보장수익률(`redemption`)을 받는다.

```json
{"tool":"convertible","flags":{"face":10000000000,"coupon-rate":0.01,"frequency":1,"maturity":3,"conversion-price":20000,"spot":18000,"vol":0.35,"rate":0.032,"spread":0.04,"steps":500}}
```

인용: `data.value`(CB 가치), `equity_component`·`debt_component`, `bond_floor`, `parity`, `premium_over_parity`, `delta`, `calculation_hash`. 봉투의 `component_method_value`는 "채권 + 무위험 콜" 분해법 값이다 — 둘의 차이가 신용위험·조기행사 효과이므로 해설에 같이 쓴다.

- 콜·풋 조항이 있으면 `call-price`/`call-start`, `put-price`/`put-start`(액면 기준 금액, 년).
- 만기보장수익률이 있으면 `redemption`(1.06 = 106%).
- 리픽싱은 모형에 없다. 전환가를 바꾼 시나리오(예: 하한 70%)를 한 번 더 돌려 범위로 보여준다.
- `assumptions`·`warnings`를 해설 끝에 옮긴다(희석·리픽싱·회수율 미반영).

## 함정
- **금액을 원 단위 정수로 바꿀 때 자릿수를 다시 센다.** "5,000억"은 `500000000000`(0이 11개)이다. 엔진은 단위를 검증하지 못하고 그대로 계산한다.


- `--basis`는 엑셀과 같다. 0=US 30/360, 1=actual/actual, 2=actual/360, 3=actual/365, 4=EU 30/360. 추측하지 말고 명시한다.
- `bond --solve price`는 액면 100 기준 **청산가격**.
- settlement·maturity·frequency를 빼면 입력 오류(종료 2). 날짜는 ISO 8601.
- 커브로 입력 채권을 재평가하면 입력 가격에 가까워야 한다. 아니면 곡선 입력을 고친다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
