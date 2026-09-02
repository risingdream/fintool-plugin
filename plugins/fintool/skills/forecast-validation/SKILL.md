---
name: forecast-validation
description: fintool MCP로 데이터 기반 재무 예측을 검증한다. 시계열 예측, rolling-origin 백테스트, 표본외 성능, seasonal naive 대비 승률, 분위수 coverage, 예측 준비도, statistical forecast, out-of-sample, forecast validation 요청에 사용한다. plan·assumption 런웨이·BP는 startup-finance, 실측 청구·수금 원장은 billing-cashflow, 실측 MRR 원장은 subscription-metrics, 회귀 prediction interval은 financial-statements로 보낸다. 숫자는 추정하지 말고 fintool 봉투만 인용한다.
---

# Forecast Validation

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"<도구>"}`로 해당 도구 하나의 최신 계약을 조회하고
`input_contract.call_example`을 복사한다. 계산은 `fintool_run {"tool":"<도구>","flags":{"<플래그>":"<값>"}}` 형태로 실행한다.
`flags`는 JSON 객체로 전달한다. `spec` 플래그를 쓰는 도구에서는 `flags.spec`도 JSON 객체로 넣고,
JSON 문자열로 이중 직렬화하지 않는다.

원격 MCP의 `timeseries`를 사용한다. 첫 실행 전에
`fintool_catalog {"tool":"timeseries"}`로 현재 `input_contract.call_example`, flags,
backtest 계약을 확인한다. in-sample `compare` RMSE로 예측을 채택하지 않는다.

수치는 성공 JSON 봉투에서만 인용한다. 필드 정본은 카탈로그와
`result.readiness`·`result.protocol`·`result.methods`·`result.origins`다.

## 요청 분류

계산 전에 요청을 하나로 분류한다. 섞어서 처리하지 않는다.

| 분류 | 뜻 | 처리 |
|---|---|---|
| `plan assumption` | 인터뷰·가정 드라이버로 미래 계획을 만든다 | `startup-finance` |
| `deterministic stress` | 명시된 가정을 바꾼 결정론 스트레스. 확률이 아니다 | `startup-finance` 또는 해당 원장 스킬 |
| `statistical forecast` | 관측 시계열에서 표본외 예측을 뽑아 품질을 판정한다 | **이 스킬** |

actual 원장을 forecast로 자동 승격하지 않는다. 실측 invoice·MRR 원장 질문은
`billing-cashflow`·`subscription-metrics`로 보낸다. 횡단면 회귀의 prediction
interval은 `financial-statements`다.

## 라우팅

| 사용자 의도 | route |
|---|---|
| 관측 시계열의 rolling-origin·coverage·준비도 | `forecast-validation` → `timeseries --mode backtest` |
| 가정 기반 런웨이·BP·캡테이블·bear/base/bull | `startup-finance` |
| 실측 청구·수금·AR·DSO | `billing-cashflow` |
| 실측 MRR·waterfall·GRR/NRR | `subscription-metrics` |
| 재무비율·듀폰·회귀 prediction interval·부도확률 | `financial-statements` |
| 경로 전체가 한 분위수일 확률, 고갈 경로 | `portfolio-risk`의 `montecarlo` |

## 라이선스와 production 범위

모델을 fintool에 내장하거나 사이드카로 실행하지 않는다. 외부 포인트·분위수는
`--forecasts` JSON으로만 받는다.

- provider, version, as_of, transform(`kind`, `make_positive`)을 숨기지 않는다.
- TimesFM-3 가중치는 비상업·비프로덕션이다. 최종 사용자 상호작용·수익 활동·제품
  MCP에 쓰지 않는다. 내부 평가 결과만 문서에 인용할 수 있다.
- TimesFM 2.5·Chronos-2 등 Apache-2.0 가중치와 사용자가 이미 가진 예측만
  production 후보로 검토한다.
- `make_positive` 기본값을 숨긴 채 적자 가능 target에 적용하지 않는다.

## 입력 확인

빠진 값을 0이나 관행값으로 채우지 않는다.

1. **분류**가 `statistical forecast`인가. 아니면 해당 스킬로 보낸다.
2. **grain** `month|quarter`와 **seasonal-lag**(월 12, 분기 4)를 명시한다.
3. **origin 이전 데이터만** 컨텍스트와 future covariate에 쓴다.
   `availability=past_future`는 `known_at_origin=true` 없이 거절된다.
4. **음수 target**: 영업이익·순이익·현금흐름은 음수가 흔하다.
   `allow-negative`와 관측 음수 개수, provider `make_positive`를 대조한다.
5. **누적 재무 흐름**: DART 분기 손익이 누적이면 `cumulative-flow`를 명시하고
   개별 기간으로 분해된 값만 예측한다. 누적 값을 그대로 넣지 않는다.
6. **결측·leading missing·구조 변화**(회계기준·연결범위)를 기록한다.
7. 외부 예측이면 provider 라이선스가 production 허용 범위인지 확인한다.

## 실행

catalog 예시의 backtest 플래그를 복사해 값만 바꾼다.

```json
{"tool":"timeseries","flags":{"mode":"backtest","grain":"quarter","seasonal-lag":4,"min-context":8,"forecast-horizon":4,"periods-per-year":4,"allow-negative":true,"make-positive":false,"cumulative-flow":false,"returns":"10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25"}}
```

- `--mode compare`는 같은 표본 RMSE라 예측 채택 근거가 아니다.
- 컨텍스트는 origin 이전 관측치 전부다. origin 번호는 1-based 마지막 학습 관측치다.
- 외부 예측은 `--forecasts`에 provider·version·transform·origin별 point/quantile을 넣는다.
- 미래 공변량은 당시 가용값만 허용한다.

## 결과 승인

숫자를 설명하기 전에 봉투를 읽는다.

- `result.readiness.ready`가 false이거나 origin이 3개 미만이면 예측을 채택하지 않는다.
- 포인트는 `result.methods[].mase`·`smape`와 `win_rate_vs_seasonal_naive`를 함께 인용한다.
  seasonal naive 대비 승률이 없으면 "더 나은 예측"이라고 쓰지 않는다.
- 분위수는 `coverage.nominal`과 `coverage.actual`을 구분해 읽는다. 명목 80%를
  실측 coverage로 말하지 않는다. 기준 미달 구간을 신뢰구간으로 말하지 않는다.
- 시점별 q10/q50/q90을 경로 시나리오 bear/base/bull 확률로 바꾸지 않는다.
  경로 확률이 필요하면 `montecarlo`다.
- 매출·비용·이익처럼 항등식이 필요한 값은 독립 예측하지 않는다. 이익은 파생하거나
  잔차를 표시한다. 모순을 숨기지 않는다.

채택·계획값 승격은 사전에 명시된 조직/사용자 정책이 있을 때만 그 기준을 적용한다.
정책이 없으면 `readiness`·MASE/sMAPE·승률·명목/실측 coverage를 보고하되 계획값으로
자동 승격하지 않는다. 이 스킬은 보편 승률·coverage 수치를 채택 기준으로 두지 않는다.

## 금지

- 검증되지 않은 예측 숫자를 만들거나 plan 값으로 자동 승격
- 사전에 명시되지 않은 보편 승률·coverage 수치로 채택·승격 판정
- actual 원장을 forecast로 복사
- in-sample `compare` RMSE로 모형 채택
- 명목 quantile을 실측 coverage로 읽기
- 시점별 분위수를 경로 시나리오 확률로 변환
- 항등식 계정의 독립 예측 후 잔차 은폐
- TimesFM-3 가중치를 production·최종 사용자 경로에 사용
- 음수 target에 숨은 `make_positive`
- origin 이후 정보·당시 비가용 공변량 사용
