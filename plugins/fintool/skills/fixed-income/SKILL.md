---
name: fixed-income
description: fintool MCP로 채권 가격·수익률·듀레이션·커브·전환사채를 계산한다. 이표채, 할인채, 국채, Z-spread, 커브 부트스트랩, 전환사채(CB)·전환옵션 가치를 요청하면 이 스킬을 쓴다. 스타트업 IR은 startup-finance, DCF는 valuation으로 보낸다. 숫자는 봉투만 인용한다.
---

# Fixed Income

원격 MCP: `fintool_catalog` → `fintool_run`.

범위: `coupon` / daycount basis → `bond` / `discount` → (선택) `curve`. 전환사채는 `bond` + `option` 분해.  
범위 밖: 주식 DCF, 스타트업 런웨이.

## 흐름

```
1. 일수 계산 기준(--basis 0~4)을 정한다
2. 이표 스케줄이 필요하면 coupon
3. 가격·수익률·듀레이션은 bond. 할인채·T-bill은 discount
4. 곡선·스팟·포워드·Z-spread는 curve
```

## 전환사채(CB)

전용 도구는 없다. **스트레이트 채권 + 전환옵션**으로 분해해 두 봉투를 합산한다. 합산은 자명한 산술이므로 계산 과정을 보여주고, 두 봉투 값을 그대로 인용한다.

```
1. 스트레이트 채권  bond --solve price  (액면 100 기준)
     yield = 무위험 + 발행사 신용스프레드   ← 전환권 없는 같은 회사 채권의 할인율
     settlement/maturity ISO 날짜, frequency 이표 횟수, rate 표면금리, basis 명시
2. 전환옵션        option --model binomial --exercise american --type call
     spot 현재 주가, strike 전환가, rate 무위험, vol 주가 변동성, maturity 잔존연수, steps 500
     dividend 가 있으면 연속 배당수익률로
3. 합산            CB = 액면 × (채권가격/100) + 전환주식수 × 옵션가격
     전환주식수 = 액면 / 전환가
4. 해설            채권 floor, 전환 패리티(주가 × 전환주식수), 프리미엄을 같이 쓴다
```

호출 예(액면 100억, 표면 1% 연 1회, 3년, 전환가 20,000, 주가 18,000, σ 35%, rf 3.2%, 스프레드 4%):

```json
{"tool":"bond","flags":{"solve":"price","settlement":"2026-08-23","maturity":"2029-08-23","rate":0.01,"yield":0.072,"redemption":100,"frequency":1,"basis":1}}
{"tool":"option","flags":{"model":"binomial","exercise":"american","solve":"price","type":"call","spot":18000,"strike":20000,"rate":0.032,"vol":0.35,"maturity":3,"steps":500}}
```

한계를 반드시 적는다: 이 분해는 옵션에 신용위험을 반영하지 않고(발행사 부도 시 전환권도 소멸), 콜·풋 조항·리픽싱·희석을 무시한다. 정밀 평가는 이항 트리에 신용스프레드를 결합한 모형(Tsiveriotis-Fernandes)이 필요하며 fintool 범위 밖이다.

## 함정

- `--basis`는 엑셀과 같다. 0=US 30/360, 1=actual/actual, 2=actual/360, 3=actual/365, 4=EU 30/360. 추측하지 말고 명시한다.
- `bond --solve price`는 액면 100 기준 **청산가격**.
- settlement·maturity·frequency를 빼면 입력 오류(종료 2). 날짜는 ISO 8601.
- 커브로 입력 채권을 재평가하면 입력 가격에 가까워야 한다. 아니면 곡선 입력을 고친다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
