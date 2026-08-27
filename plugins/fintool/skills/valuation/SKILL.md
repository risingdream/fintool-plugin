---
name: valuation
description: fintool MCP로 WACC·DCF·배수 가치평가를 한다. 내재가치, 기업가치, 주당가치, 할인율, 영구성장률, 민감도 격자, 삼성전자형 DCF를 요청하면 이 스킬을 쓴다. 재무비율·듀폰 분해·EPS·FCFF 산출·부도확률은 financial-statements, 펀드 성과귀속·샤프·PE 펀드지표는 fund-performance, 최적 비중·VaR는 portfolio-risk, 채권 가격·듀레이션은 fixed-income, 스타트업 런웨이·캡테이블·IR HTML은 startup-finance, 옵션 가격·그릭스·내재변동성은 option으로 보낸다. 숫자는 fintool 봉투만 인용한다.
---

# Valuation

도구는 **원격 MCP**다. `fintool_catalog` → `fintool_run`. 로컬 바이너리를 설치하지 않는다.
`fintool_catalog`는 인자 없이 부르면 목록만 준다. 플래그는 `{"tool": "wacc"}`처럼 도구를 지정해 받는다.

범위: 가정 → `wacc` → `dcf` → (선택) `equity`.  
범위 밖: 재무비율·듀폰·EPS·회계 정규화·부도확률(`financial-statements` — FCFF **산출**은 거기서 하고 여기서는 받아 쓴다), 성과귀속·펀드지표(`fund-performance`), 최적 비중·VaR(`portfolio-risk`), 채권 프라이싱(`fixed-income`), 런웨이·BP·캡테이블·IR HTML(`startup-finance`), 옵션 가격·그릭스·내재변동성(`option`).

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

**단, `workflow` 경로에서는 `--from`을 쓸 수 없다.** `--from`은 계산 커맨드의 플래그를 채우는데
`workflow run`에는 채울 플래그가 없다(`unknown_input`). 출처 있는 입력이 필요하면 둘 중 하나다.

1. `wacc`·`dcf`를 **개별 커맨드로 부른다.** 각 단계 봉투가 자기 출처를 싣는다. (기본)
2. workflow가 꼭 필요하면 `fetch`로 값과 출처를 먼저 받아 spec에 값으로 박고,
   **출처는 봉투가 아니라 해설에서 인용한다.** 그 실행의 봉투에는 `sources`가 없다.

## 출처 있는 입력을 먼저 쓴다

무위험수익률·시가총액·주가처럼 **공개 소스에 있는 값은 사용자에게 묻기 전에 `--from`으로 받는다.**
받은 값은 봉투 `sources[]`에 provider·기준일·확인 URL과 함께 실린다. 손으로 친 주석("2026-07-24 국고채 10년물")은
그 숫자가 그날 그 값이었다는 것을 아무것도 보증하지 않는다 — 봉투가 말하게 한다.

### 어떤 입력에 어떤 ref를 붙이나

| 도구 | 입력 | ref | 주의 |
|---|---|---|---|
| `wacc` | `risk-free-rate` | `ecos:rate:ktb10y` | 예측기간이 짧으면 `ktb3y`·`ktb5y`. 만기를 현금흐름 기간에 맞춘다 |
| `wacc` | `cost-of-debt` | `ecos:rate:corp-aa-` | 회사채 3년 AA- **시장금리**다. 그 회사의 실제 조달금리를 알면 그쪽이 낫다 |
| `wacc` | `equity-value` | `krx:marketcap:005930` | 시가총액. 장부가가 아니다 |
| `equity` | `price` | `krx:price:005930` | 종가 |
| `equity` · `dcf` | `shares` | `dart:shares:005930:2025:FY` | **보통주 유통주식수(1차 공시, 자기주식 제외)**. 발행주식총수가 필요하면 뒤에 `:issued` |
| `equity` · `dcf` | `shares` | `krx:shares:005930` | 상장주식수. 기준일 기준이라 공시와 시점이 다를 수 있다 |

**주식 수를 언론 보도나 기억에서 쓰지 않는다.** `dart:shares`가 사업보고서 원문(접수번호 포함)에서
직접 받는다. `data.by_kind`에 보통주·우선주·합계가 다 있는데, **합계를 P/E 분모로 쓰면 안 된다** —
우선주가 포함돼 보통주 주가와 단위가 맞지 않는다. 실제로 더존비즈온 2025년은 합계 발행 `31,477,993`주,
보통주 발행 `30,382,784`주로 3.6% 차이가 난다.

**기본값은 유통주식수다.** 같은 회사의 보통주 유통주식수는 `28,028,674`주로, 발행총수와
자기주식 `2,354,110`주만큼 다르다. 배수의 분모는 유통주식수 쪽이고 봉투의 `basis`가
어느 쪽인지 말해준다 — 그 값을 답에 함께 적는다.

`beta`·`market-risk-premium`·`fcf`·`terminal-growth`에는 ref가 없다. 이 값들은 여전히 사용자 입력이거나
「입력이 빠졌을 때」의 가정이다.

### 호출 형태 — 플래그만 쓴다

원격 MCP는 위치 인자를 받지 않는다. `--from`은 루트 persistent 플래그라 어느 계산 도구에서든 같은 문법이다.

```json
{"tool":"wacc","flags":{
  "from":["risk-free-rate=ecos:rate:ktb10y@2026-08-21","equity-value=krx:marketcap:005930"],
  "beta":1.1,"market-risk-premium":0.055,"cost-of-debt":0.05,
  "debt-value":12000000000000,"tax-rate":0.22}}
```

값만 확인하고 싶으면 `fetch`를 부른다. 여기서도 위치 인자가 아니라 `ref` 플래그다.

```json
{"tool":"fetch","flags":{"ref":"ecos:rate:ktb10y"}}
```

- **`@as_of`를 생략하면 "최신"이고 최신은 재현되지 않는다.** 리포트로 남길 계산이면 첫 호출의
  `sources[].as_of`를 ref에 박아 다시 부른다(`ecos:rate:ktb10y@2026-08-21`).
- 같은 입력에 `--from`과 값 플래그를 같이 주면 `conflicting_input`으로 거절한다. 둘 중 하나만 준다.
- 좌변을 생략하면 스칼라 ref는 `missing_target`이다. `--from <입력>=<ref>` 형태로 준다.
- 금액 자리에 금리 ref를 겨누면 `source_type_mismatch`로 걸린다. 오류 메시지가 기대 타입을 말해준다.

### 출처를 어떻게 인용하나

`sources[]`의 `provider`·`as_of`·`url`을 **답과 리포트에 함께 적는다.** ECOS 이용약관 제7조②가
"ECOS에서 제공된 정보임을 표시"할 의무를 지운다 — 인용은 예의가 아니라 이용 조건이다.

> 무위험수익률 3.854% — 한국은행 ECOS 국고채(3년), 기준일 2026-08-21 (`ecos.bok.or.kr`)
> 시가총액 — 금융위원회 주식시세정보(원천 KRX), 기준일 2026-08-21 (`data.krx.co.kr`)

- `sources[].fields`에 이름이 있는 입력만 자동 취득이다. **거기 없는 입력은 사람이 준 값이거나 내가 정한 가정이고,
  그 둘을 한 줄에 섞어 적지 않는다.**
- DART 값을 인용할 때는 "공시정보의 정확성·완전성은 보장되지 않는다"(DART 이용약관 제23조)를 각주 한 줄로 함께 낸다.
- `sources[].cached`가 `true`면 이번 실행이 파일 캐시에서 왔다는 뜻이다. `as_of`는 그대로 유효하다.
- ref 여러 개를 쓰면 `sources`도 여러 개다. 각 원소의 `fields`가 자기가 채운 입력을 말한다.
- **출처는 그 값을 받은 단계의 봉투에만 붙는다.** `wacc`가 ECOS에서 무위험수익률을 받아도
  그 wacc를 넘겨받은 `dcf` 봉투에는 `sources`가 없다 — dcf의 입력은 전부 수기이기 때문이다.
  **최종 리포트에는 각 단계 봉투의 `sources`를 모아서 적는다.** 마지막 봉투만 보고 "출처 없음"으로 쓰지 않는다.

### 키가 없거나 소스가 막혔을 때

인증키가 없으면 `missing_credential`, 한도 초과면 `quota_exceeded`, 없는 데이터면 `source_not_found`로 떨어진다.
**어느 경우에도 계산을 포기하지 않는다.**

1. 오류 메시지에 조회 화면 URL과 대체할 수기 플래그가 들어 있다. 그것을 사용자에게 그대로 전달한다.
2. 사용자가 값을 주면 값 플래그로 계산한다. **자동 취득과 수기 입력은 `data`가 바이트 단위로 같다** —
   `sources`는 `calculation_hash` 밖에 있으므로 해시도 같다. 결과가 달라지지 않는다.
3. 값도 못 받으면 「입력이 빠졌을 때」의 기본값 절차로 내려간다. 그때는 `assumptions[].source`에
   `기본값 제안 YYYY-MM-DD (사용자 미제공)`으로 남기고, 그 실행의 봉투에는 `sources`가 없다 — 정직한 상태다.
4. `quota_exceeded`는 재시도하지 않는다. 같은 호출이 같은 이유로 다시 실패한다.

## 입력이 빠졌을 때

「출처 있는 입력을 먼저 쓴다」로 채울 수 있는 값을 먼저 채운 뒤 이 절로 온다.
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

## 배수는 필요한 지표만 고른다 (`equity --model multiples`)

P/E·P/B·P/S만 필요한데 PEG와 EV 배수까지 계산하면 `--earnings-growth`(미래 추정치)와
`--ev-metric`·`--net-debt`를 모르는 채로 채워야 하고, **그 근거 없는 숫자로 만든 배수가 결과에 남는다.**
`--metrics`로 고르면 고른 지표가 쓰는 입력만 필수가 된다.

```json
{"tool":"equity","flags":{"model":"multiples","method":"observed","metrics":"pe,pb,ps",
  "price":163000,"eps":3982,"book-value-per-share":21500,"sales-per-share":14700}}
```

- 고를 수 있는 지표: `pe` `pb` `ps` `peg` `ev`
- 고르지 않은 지표는 결과에서 **사라지고** `omitted_metrics`에 이유가 남는다. 0으로 채워지지 않는다
- `--metrics`를 생략하면 지금까지처럼 전부 계산한다(입력도 전부 필수)
- `ev`를 고르면 `net_debt`가 결과에 함께 나온다. **순부채를 모른 채 0을 준 `enterprise_value`는
  주식가치와 같은 값이므로 그대로 인용하면 오독이다** — 모르면 `ev`를 고르지 않는다

필수 플래그가 빠지면 **한 번에 전부** 알려준다. 하나씩 채워 네 번 왕복하지 않는다.

```
--book-value-per-share, --earnings-growth, --eps, --ev-metric, --net-debt, --sales-per-share,
--shares를 지정해야 합니다 (필수 10개 중 7개 누락)
```

## 카탈로그 호출 줄이기

`fintool_catalog`는 스펙 조회일 뿐 계산이 아니다. 계측에서 전체 도구 호출의 31%가 카탈로그였다.

- 이 문서에 **호출 예시가 있는 도구는 카탈로그를 부르지 않는다.** 예시를 그대로 쓰고 숫자만 바꾼다.
- 예시가 없는 도구만 `fintool_catalog {"tool":"<이름>"}`으로 그 도구 하나를 받는다. 인자 없는 전체 목록은 어떤 도구가 있는지 모를 때만 쓴다.
- 호출이 실패하면 오류 봉투가 유효 플래그 목록과 `input_contract`를 함께 준다. 그것을 보고 고치면 되고 카탈로그를 따로 부를 필요가 없다.

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
