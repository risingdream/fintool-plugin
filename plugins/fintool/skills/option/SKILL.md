---
name: option
description: fintool MCP로 옵션 가격·그릭스·내재변동성을 계산한다. 콜옵션, 풋옵션, Black-Scholes, 이항트리, 그릭스(delta/gamma/vega), implied vol, 내재변동성 역산, 변동성 스마일, 풋-콜 패리티, 커버드콜·스트래들을 요청하면 이 스킬을 쓴다. 연율·실현 변동성·VaR·최적 비중은 portfolio-risk, 전환사채 전환옵션은 fixed-income, DCF·WACC는 valuation으로 보낸다. 숫자는 봉투만 인용한다. 로컬 Python·SciPy로 역산하지 않는다.
---

# Option

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

원격 MCP: `fintool_catalog` → `fintool_run`. 로컬 바이너리·Python·SciPy로 계산하지 않는다.

범위: `option` (`price` / `greeks` / `iv` / `payoff` / `parity` / `strategy`).
범위 밖: 실현·연율 변동성·VaR·최적 비중(`portfolio-risk`), 전환사채 전환옵션(`fixed-income`).

## 흐름

```
1. fintool_catalog {"tool":"option"} 으로 플래그·예시를 확인한다
2. fintool_run {"tool":"option","flags":{...}} 으로 계산한다
3. 성공 봉투의 값과 calculation_hash만 인용한다. 해시는 자르지 않는다
```

카탈로그를 건너뛰고 로컬에서 역산하지 않는다. `vol` 도구는 수익률 시계열의 실현 변동성이며 옵션 IV가 아니다.

## 내재변동성 — 최소 성공 예

시장가격에서 IV를 역산한다. `--vol`을 주면 거절된다. `--price`가 필수다. 만기는 연 단위 소수(`0.5` = 6개월).

```json
{"tool":"option","flags":{"solve":"iv","type":"call","spot":100,"strike":90,"rate":0.03,"maturity":0.5,"price":13.2}}
```

행사가가 여러 개면 배치 한 번이다.

```json
{"tool":"option","flags":{"solve":"iv","type":"call","spot":100,"rate":0.03,"maturity":0.5,"batch":{"cases":[{"label":"K90","params":{"strike":90,"price":13.2}},{"label":"K100","params":{"strike":100,"price":6.8}},{"label":"K110","params":{"strike":110,"price":3.1}}]}}}
```

인용: 단건은 `data.implied_vol`, 배치는 `data.results[].implied_vol`. 해설은 봉투 값만 쓴다.

## 가격·그릭스

```json
{"tool":"option","flags":{"solve":"price","type":"call","spot":100,"strike":100,"rate":0.03,"vol":0.2,"maturity":0.5}}
```

```json
{"tool":"option","flags":{"solve":"greeks","type":"call","spot":100,"strike":105,"rate":0.03,"vol":0.2,"maturity":0.5}}
```

## 함정

- 로컬 Python/SciPy/`brentq`로 IV를 풀지 않는다. 그 값은 fintool 봉투가 아니다.
- `--solve iv`에 `--vol`을 넣지 않는다.
- 비율은 소수다(3% = `0.03`).
- 일수로 주려면 `--maturity-days`다. `--maturity`와 동시에 주면 입력 오류다.
