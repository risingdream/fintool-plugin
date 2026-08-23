---
name: fixed-income
description: fintool MCP로 채권 가격·수익률·듀레이션·커브를 계산한다. 이표채, 할인채, 국채, Z-spread, 커브 부트스트랩을 요청하면 이 스킬을 쓴다. 스타트업 IR은 startup-finance, DCF는 valuation으로 보낸다. 숫자는 봉투만 인용한다.
---

# Fixed Income

원격 MCP: `fintool_catalog` → `fintool_run`.

범위: `coupon` / daycount basis → `bond` / `discount` → (선택) `curve`.  
범위 밖: 주식 DCF, 스타트업 런웨이.

## 흐름

```
1. 일수 계산 기준(--basis 0~4)을 정한다
2. 이표 스케줄이 필요하면 coupon
3. 가격·수익률·듀레이션은 bond. 할인채·T-bill은 discount
4. 곡선·스팟·포워드·Z-spread는 curve
```

## 함정

- `--basis`는 엑셀과 같다. 0=US 30/360, 1=actual/actual, 2=actual/360, 3=actual/365, 4=EU 30/360. 추측하지 말고 명시한다.
- `bond --solve price`는 액면 100 기준 **청산가격**.
- settlement·maturity·frequency를 빼면 입력 오류(종료 2). 날짜는 ISO 8601.
- 커브로 입력 채권을 재평가하면 입력 가격에 가까워야 한다. 아니면 곡선 입력을 고친다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
