---
name: fixed-income
description: fintool MCP로 채권 가격·수익률·듀레이션·커브·전환사채를 계산한다. 이표채, 할인채, 국채, 컨벡시티, Z-spread, 커브 부트스트랩·스팟·포워드, 전환사채(CB)·전환옵션 가치를 요청하면 이 스킬을 쓴다. 시장가격에서 뽑는 Z-spread는 여기지만, 재무제표로 추정하는 Merton 부도확률·신용스프레드는 financial-statements다. 채권 포트폴리오 성과귀속은 fund-performance, DCF·WACC는 valuation, 최적 비중·VaR는 portfolio-risk, 스타트업 IR은 startup-finance로 보낸다. 숫자는 봉투만 인용한다.
---

# Fixed Income

원격 MCP: `fintool_catalog` → `fintool_run`.

범위: `coupon` / daycount basis → `bond` / `discount` → (선택) `curve`. 전환사채는 `convertible`.  
범위 밖: 재무제표 기반 신용위험 — Merton 부도확률·구조모형 신용스프레드(`financial-statements`). 듀레이션 기반 채권 성과귀속(`attribution --model fixed-income`)은 `fund-performance`. 주식 DCF(`valuation`), 최적 비중·VaR(`portfolio-risk`), 스타트업 런웨이(`startup-finance`).

## 흐름

```
1. 일수 계산 기준(--basis 0~4)을 정한다
2. 이표 스케줄이 필요하면 coupon
3. 가격·수익률·듀레이션은 bond. 할인채·T-bill은 discount
4. 곡선·스팟·포워드·Z-spread는 curve
```

## 출처 있는 입력을 먼저 쓴다

국고채 커브와 시장금리는 **사용자에게 묻기 전에 `--from`으로 받는다.** 받은 값은 봉투 `sources[]`에
provider·기준일·확인 URL과 함께 실린다. 곡선을 손으로 받아 적으면 그 숫자가 그날 그 값이었다는 것을
아무것도 보증하지 않는다.

### 어떤 입력에 어떤 ref를 붙이나

| 도구 | 입력 | ref | 주의 |
|---|---|---|---|
| `curve` | `maturities` + `yields` | `ecos:curve:ktb` | **좌변을 생략한다.** 한 ref가 두 입력을 함께 채운다. 국고채 1·2·3·5·10·20·30·50년 |
| `bond` | `yield` | `ecos:rate:ktb3y` (`ktb1y`~`ktb50y`) | 평가 대상 채권의 잔존만기에 맞춘 구간을 고른다 |
| `bond` · `curve` | 비교 곡선 | `ecos:rate:corp-aa-` · `ecos:rate:corp-bbb-` · `ecos:rate:cd91` · `ecos:rate:cp91` | Z-spread 기준 곡선 |
| — | 기준금리 | `ecos:rate:base` | 해설용. 채권 프라이싱 입력이 아니다 |

`convertible`의 `--spread`(발행사 신용스프레드)와 `--vol`에는 ref가 없다. 사용자 입력이거나 명시한 가정이다.

### ⚠️ `convertible --rate`·`credit --rate`에 ECOS 값을 그대로 꽂지 않는다

이 둘의 `--rate`는 **연속복리** 자리이고 ECOS가 주는 국고채 수익률은 연 단위 관행 수익률이다.
`--from rate=ecos:rate:ktb3y`는 오류 없이 통과하지만 규약이 다른 숫자가 들어간다 — 엔진은 못 잡는다.
환산은 `curve`를 거친다.

```json
{"tool":"curve","flags":{"solve":"nelson-siegel","from":["ecos:curve:ktb@2026-08-21"],
                         "compounding":"continuous","evaluate":"3"}}
```

`data.curve[].rate`가 연속복리 스팟이다. 그 값을 `convertible`·`credit`의 `--rate`에 넣고,
해설에 "ECOS 국고채 커브(기준일 …)를 연속복리 스팟으로 환산했다"고 적는다.

**출처는 그 값을 받은 단계의 봉투에만 붙는다.** `curve` 봉투에는 ECOS `sources`가 있지만,
그 결과를 수기로 넘겨받은 `convertible` 봉투에는 없다. 최종 리포트에는 **두 단계 봉투를 모아서**
인용한다 — 마지막 봉투만 보고 "출처 없음"으로 쓰지 않는다.

### 호출 형태 — 플래그만 쓴다

원격 MCP는 위치 인자를 받지 않는다. `--from`은 루트 persistent 플래그라 어느 계산 도구에서든 같은 문법이다.

```json
{"tool":"bond","flags":{"solve":"price","from":["yield=ecos:rate:ktb3y@2026-08-21"],
                        "settlement":"2026-08-21","maturity":"2029-08-21",
                        "rate":0.04,"frequency":2,"basis":1}}
```

값만 확인하고 싶으면 `fetch`를 부른다. 여기서도 위치 인자가 아니라 `ref` 플래그다.

```json
{"tool":"fetch","flags":{"ref":"ecos:curve:ktb"}}
```

- **`@as_of`를 생략하면 "최신"이고 최신은 재현되지 않는다.** 리포트로 남길 계산이면 첫 호출의
  `sources[].as_of`를 ref에 박아 다시 부른다.
- **휴일·주말은 값이 없다.** 어댑터가 최대 7일 되짚어 직전 영업일을 쓰고 실제 기준일을 `as_of`에 적는다.
  요청한 날짜와 `as_of`가 다를 수 있으므로 **인용은 반드시 봉투의 `as_of`로 한다.**
- 같은 입력에 `--from`과 값 플래그를 같이 주면 `conflicting_input`으로 거절한다.
- `ecos:curve:ktb`에 좌변을 붙이면(`--from yields=ecos:curve:ktb`) 두 입력 중 하나만 채워져 곡선이 깨진다. 붙이지 않는다.

### 출처를 어떻게 인용하나

`sources[]`의 `provider`·`as_of`·`url`을 **답과 리포트에 함께 적는다.** ECOS 이용약관 제7조②가
"ECOS에서 제공된 정보임을 표시"할 의무를 지운다 — 인용은 예의가 아니라 이용 조건이다.

> 국고채 3년 3.854% — 한국은행 ECOS 시장금리(817Y002), 기준일 2026-08-21 (`ecos.bok.or.kr`)

- `sources[].note`에 통계표·항목 코드가 들어 있다(`통계표 817Y002 / 항목 010200000`). 어느 계열을 썼는지 밝힐 때 그대로 쓴다.
- `sources[].fields`에 이름이 있는 입력만 자동 취득이다. 거기 없는 입력은 사람이 준 값이거나 내가 정한 가정이고, 그 둘을 한 줄에 섞어 적지 않는다.

### 키가 없거나 소스가 막혔을 때

인증키가 없으면 `missing_credential`, 한도 초과면 `quota_exceeded`로 떨어진다. **계산을 포기하지 않는다.**

1. 오류 메시지에 조회 화면 URL과 대체할 수기 플래그가 들어 있다. 사용자에게 그대로 전달한다.
2. 사용자가 값을 주면 값 플래그로 계산한다. **자동 취득과 수기 입력은 `data`가 바이트 단위로 같다** —
   `sources`는 `calculation_hash` 밖이라 해시도 같다. 결과가 달라지지 않는다.
3. 그 실행의 봉투에는 `sources`가 없다. 그러면 해설에도 출처를 적지 않는다 — 봉투가 모르는 것을 문장이 아는 척하지 않는다.
4. `quota_exceeded`는 재시도하지 않는다. `ECOS_API_KEY`가 `sample` 키면 **커브 조회는 10건 한도에 걸려 되지 않는다**(`ERROR-301`).
   정식 키가 필요하고, 그동안은 만기별 `ecos:rate:ktb*`를 따로 받거나 수기 입력으로 간다.

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

## 카탈로그 호출 줄이기

`fintool_catalog`는 스펙 조회일 뿐 계산이 아니다. 계측에서 전체 도구 호출의 31%가 카탈로그였다.

- 이 문서에 **호출 예시가 있는 도구는 카탈로그를 부르지 않는다.** 예시를 그대로 쓰고 숫자만 바꾼다.
- 예시가 없는 도구만 `fintool_catalog {"tool":"<이름>"}`으로 그 도구 하나를 받는다. 인자 없는 전체 목록은 어떤 도구가 있는지 모를 때만 쓴다.
- 호출이 실패하면 오류 봉투가 유효 플래그 목록과 `input_contract`를 함께 준다. 그것을 보고 고치면 되고 카탈로그를 따로 부를 필요가 없다.

## 함정
- **금액을 원 단위 정수로 바꿀 때 자릿수를 다시 센다.** "5,000억"은 `500000000000`(0이 11개)이다. 엔진은 단위를 검증하지 못하고 그대로 계산한다.


- `--basis`는 엑셀과 같다. 0=US 30/360, 1=actual/actual, 2=actual/360, 3=actual/365, 4=EU 30/360. 추측하지 말고 명시한다.
- `bond --solve price`는 액면 100 기준 **청산가격**.
- settlement·maturity·frequency를 빼면 입력 오류(종료 2). 날짜는 ISO 8601.
- 커브로 입력 채권을 재평가하면 입력 가격에 가까워야 한다. 아니면 곡선 입력을 고친다.
- 숫자를 만들지 않는다. HTML을 쓰지 않는다.
