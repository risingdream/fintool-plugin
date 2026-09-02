# 사업 유형별 드라이버 체인

`financial-model` 드라이버 스펙으로 옮길 때 참고하는 6종이다. 드라이버 계층은 `cash`·`accrual` 두 모드가 공유하므로 여기 규약은 모드와 무관하다. **템플릿이 아니라 출발점**이다. 실제 사업은 대개 섞여 있고, 인터뷰에서 나온 실제 경로가 항상 우선한다.

실행 가능한 전체 스펙은 `fintool_catalog {"tool":"financial-model"}`의
`input_contract.call_example`을 복사하고, 아래 드라이버 체인으로 실제 값만 교체해 만든다.

## 유형별 모드 권고

기본은 항상 cash다. 아래는 **accrual로 갈 이유가 유형 자체에 있는가**를 정리한 것이지, 처음부터 accrual로 시작하라는 뜻이 아니다.

| 유형 | 권고 | 이유 |
|---|---|---|
| 구독 SaaS | cash | 선불 정산이면 인식과 현금이 같은 달이다. 연간 선결제가 크면 `deferred_revenue` 때문에 accrual |
| 성공보수 | cash | 인용 시점에 정산된다. 회수가 몇 달 밀리면 accrual |
| 마켓플레이스 | cash | 정산 주기가 길거나(에스크로) 선정산을 하면 accrual |
| 커머스 | **accrual** | 재고가 현금을 먼저 잡아먹는다. cash는 재무상태표를 내지 않아 재고 잔액이 남지 않는다 |
| 광고 | cash | 대행사를 끼면 회수가 60~90일이라 accrual |
| 하드웨어 + 소모품 | **accrual** | 부품 재고 + 생산 설비 capex. capex가 모드에 따라 완전히 다르게 처리된다 |

`statement_role`은 모드와 무관하게 **커머스·하드웨어에서 반드시 명시한다.** 매출에 연동되는 원가가 있는데 생략하면 매출총이익률이 100%로 나온다.

---

## 1. 구독 SaaS

**체인**: 고객 기반 × ARPA = 매출

| 드라이버 | schedule | 비고 |
|---|---|---|
| `customers` | `cohort` (`initial`, `retention`, `additions`) | 전월 잔량이 이월된다. `monthly` 배열로 손계산하지 않는다 |
| `arpa` | `monthly` | 인상 계획이 있으면 `growth` |
| `gross_margin` | `monthly` | |
| `cac` | `monthly` | |

```json
{"id": "customers", "value_type": "count", "unit": "customer",
 "schedule": {"kind": "cohort", "initial": 100, "retention": 0.9, "additions": 30}}
```

유닛 이코노믹스는 `subscription`. `churn-rate`는 `1 - retention`과 같은 값이어야 한다 — 어긋나면 창업자에게 어느 쪽이 맞는지 묻는다.

`metric`으로 노출할 것: 활성 고객 수.

---

## 2. 성공보수 (환급·정산 대행)

**체인**: 조회 × 판정률 × 신청률 × 인용률 = 인용건수 → × 평균환급액 × 수수료율 = 매출

| 드라이버 | schedule | 비고 |
|---|---|---|
| `lookups` | `monthly` 또는 `growth` | 유입 |
| `eligible_rate` `apply_rate` `approve_rate` | `monthly` | 퍼널 각 단계 |
| `ticket_low/mid/high` + 각 `_share` | `monthly` | 환급액 분포. 단일 평균값보다 정확하다 |
| `fee_rate` | `monthly` | |
| `fee_cap` `fee_floor` | `monthly` | 상한·최소수수료 |
| `partner_share` | `monthly` | 제휴 세무법인 배분 |
| `processing_cost` | `monthly` | 건당 검토 원가 |

**퍼널 각 단계를 `metric` line item으로 낸다.** 매출 하나의 중첩 expression에 밀어넣으면 월별 건수가 결과에서 사라진다.

가중평균 환급액:

```json
{"op": "add", "args": [
  {"op": "add", "args": [
    {"op": "multiply", "args": [{"ref": "ticket_low"}, {"ref": "ticket_low_share"}]},
    {"op": "multiply", "args": [{"ref": "ticket_mid"}, {"ref": "ticket_mid_share"}]}]},
  {"op": "multiply", "args": [{"ref": "ticket_high"}, {"ref": "ticket_high_share"}]}]}
```

수수료 상한:

```json
{"op": "min", "args": [{"op": "multiply", "args": [{"ref": "refund"}, {"ref": "fee_rate"}]}, {"ref": "fee_cap"}]}
```

주의할 것 세 가지:

- **재이용이 없다.** LTV를 쓰지 말고 `unit-economics --model transaction`으로 건당 공헌이익과 손익분기 건수를 낸다.
- **고액 구간일수록 자동 판정이 안 된다.** 건당 처리원가를 구간별로 다르게 잡아야 한다. 전 구간 같은 원가를 쓰면 고액 시나리오가 과대평가된다.
- **제휴 배분은 협상 가능한 비용이 아니라 구조적 비용**인 경우가 많다(세무대리 업무 제한 등). 창업자에게 확인한다.

---

## 3. 마켓플레이스

**체인**: 거래건수 × AOV = GMV → × take rate = 매출

| 드라이버 | schedule |
|---|---|
| `orders` | `growth` 또는 `cohort`(재구매 있으면) |
| `aov` | `monthly` |
| `take_rate` | `monthly` |
| `payment_fee_rate` | `monthly` |

`metric`으로 노출할 것: GMV. 투자자가 가장 먼저 묻는 숫자이고 매출과 다르다.

양면 시장이면 공급자 수도 드라이버로 두고, 거래건수를 `공급자 × 공급자당 거래`로 분해할지 창업자와 정한다.

---

## 4. 커머스

**체인**: 수량 × 단가 = 매출, 매출 × 원가율 = 매출원가

| 드라이버 | schedule |
|---|---|
| `units` | `growth` |
| `price` | `monthly` |
| `cogs_rate` | `monthly` (규모 효과가 있으면 연도별로 낮춘다) |
| `return_rate` | `monthly` |
| `shipping_cost` | `monthly` |

반품률을 빠뜨리기 쉽다. 매출에서 차감할지 비용으로 잡을지 회계 처리를 정하고 `assumptions`에 남긴다.

재고 잔액 롤포워드가 필요하면 `--mode accrual`을 쓴다. `cash` 모드는 재무상태표를 내지 않아 재고 잔액이 남지 않는다.

---

## 5. 광고

**체인**: 노출 × CTR × CPC = 매출

| 드라이버 | schedule |
|---|---|
| `impressions` | `growth` |
| `ctr` | `monthly` |
| `cpc` 또는 `cpm` | `monthly` |
| `fill_rate` | `monthly` |

`metric`으로 노출할 것: 클릭 수, eCPM.

계절성이 크면 `monthly.values`에 12개 값을 주는 게 맞다 — 이건 손계산이 아니라 실제로 월마다 다른 값이다.

---

## 6. 하드웨어 + 소모품

**체인**: 기기 판매 + 설치기반 × 소모품 단가

| 드라이버 | schedule | 비고 |
|---|---|---|
| `devices_sold` | `growth` | |
| `device_price` | `monthly` | |
| `installed_base` | `cohort` (`retention`으로 이탈 반영) | 누적 설치 대수 |
| `consumable_arpu` | `monthly` | 대당 월 소모품 매출 |
| `device_cogs_rate` | `monthly` | 기기는 마진이 낮거나 역마진일 수 있다 |

기기와 소모품을 **다른 `subtype`으로 분리**한다(`hardware`, `consumable`). 마진 구조가 완전히 달라 합치면 의사결정을 못 한다.

`installed_base`의 `additions`는 `devices_sold`와 같은 값이어야 하는데, 드라이버는 서로를 참조하지 못한다. 두 곳에 같은 값을 넣고 `assumptions`에 그 사실을 적는다.

---

## 섞여 있을 때

대부분의 실제 사업이 여기 해당한다.

- 매출원별로 line item을 나누고 `subtype`을 다르게 준다
- 각 매출원의 드라이버 체인을 독립적으로 세운다
- 공통 드라이버(고정비·인건비·CAC)는 하나만 둔다
- 시나리오는 **가장 불확실한 매출원의 드라이버**를 축으로 짠다
