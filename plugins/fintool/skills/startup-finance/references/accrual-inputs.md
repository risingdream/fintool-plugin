# accrual 모드 입력 블록

`--mode accrual`일 때만 쓰는 블록 7종이다. **전부 선택이고, cash 모드는 있어도 무시한다.** 그래서 같은 스펙을 두 모드로 다 돌릴 수 있다.

안 준 블록은 0으로 채우되 엔진이 `default_applied` 경고를 낸다. **그 경고를 사용자에게 그대로 전달한다.** 자동으로 채운 0과 창업자가 확인한 0은 다르다.

---

## 두 모드의 차이는 어디서 오나

| 원천 | 왜 갈리나 |
|---|---|
| 법인세 | accrual만 EBIT에 세율을 곱해 현금을 뺀다 |
| 감가상각 | 손익에는 들어가고 현금에는 안 들어간다. capex 자체는 두 모드 모두 유출이라 상쇄된다 |
| 운전자본 | 인식 시점과 현금 시점의 차이. AR·재고·AP를 주지 않으면 0이라 차이도 0 |

**셋 다 없으면 기말현금이 정확히 일치한다.** 안 맞으면 스펙을 다시 본다.

실측(동일 경제, 12개월, 매출 2천만/월, 비용 1,500만/월, 1월 capex 1.2억, 기초현금 5억):

```
cash    기말현금 440,000,000   영업이익 60,000,000
accrual 기말현금 432,080,000   순이익   28,080,000
차이     7,920,000 = 법인세 전액 (EBIT 36,000,000 × 22%), 잔차 0
```

감가상각 2,400만은 현금 차이에 기여하지 않는다. capex 1.2억은 cash에서 투자현금 유출, accrual에서 자산화되지만 **현금으로는 두 모드 모두 같은 달에 나간다.**

적자 구간에서는 법인세가 0이라 **두 모드가 완전히 일치한다.** 흑자로 돌아서는 달부터 갈리기 시작한다.

---

## opening_balance_sheet — 기초 대차대조표

| 필드 | 뜻 |
|---|---|
| `cash` | 기초 현금 |
| `accounts_receivable` | 매출채권 |
| `inventory` | 재고자산 |
| `other_current_assets` | 기타유동자산 |
| `ppe_net` | 유형자산 순장부가 |
| `accounts_payable` | 매입채무 |
| `deferred_revenue` | 선수수익 |
| `other_current_liabilities` | 기타유동부채 |
| `debt` | 차입금 |
| `contributed_capital` | 자본금 |
| `retained_earnings` | 이익잉여금 |

### 자동 균형 — "모른다"와 "0이다"는 다르게 다룬다

자본 두 항목(`contributed_capital`·`retained_earnings`)을 **생략하면** 자산과 같은 금액을 `contributed_capital`로 넣어 균형을 맞추고 경고를 낸다.

```
opening_balance_sheet: 자본을 주지 않아 자산과 같은 금액을 contributed_capital로
넣어 균형을 맞췄습니다
```

**0을 명시하면 의도로 보고 불균형을 그대로 거부한다.** 창업자가 "자본금 0"이라고 답한 것과 아무도 묻지 않은 것을 구분하기 위해서다. 그래서 이 두 필드만 스키마에서 포인터다.

기초 현금만 알고 나머지를 모르면 `cash`만 주고 나머지를 생략한다. 억지로 숫자를 지어내지 않는다.

---

## working_capital — 운전자본 월별 잔액

`accounts_receivable` · `inventory` · `other_current_assets` · `accounts_payable` · `deferred_revenue` · `other_current_liabilities` 6개 배열. 각 배열은 길이 1(전 기간 동일) 또는 `months`.

**잔액을 준다. 회수일수(DSO)가 아니다.** 창업자가 "60일 회수"라고 답하면 월매출 × 2 정도로 환산해 잔액을 만들고, 그 환산 근거를 `assumptions`에 남긴다.

여섯 개가 전부 비면:

```
working_capital: 전부 0으로 채웠습니다. 매출·비용이 인식과 동시에 현금으로
정산된다고 가정한 것입니다
```

이 가정이 실제로 맞는 사업도 있다(선불 구독, 즉시 정산 서비스). **맞다면 경고를 전달하면서 "이 가정이 맞다"고 함께 적는다.** 틀린데 방치하면 현금 시점이 통째로 어긋난다.

---

## ppe — 유형자산

| 필드 | 뜻 |
|---|---|
| `capex` | 월별 투자 (배열). `statement_role: capex` line item이 있으면 거기서 집계된다 |
| `existing_depreciation_amortization` | 기존 자산의 월별 감가상각 (배열) |
| `useful_life_months` | 신규 capex의 내용연수 (정수). 정액법 |

신규 capex는 `useful_life_months`로 정액 상각한다. 기존 자산 상각은 별도로 준다 — 엔진이 과거 취득가를 모르기 때문이다.

---

## debt — 차입

`draws` · `repayments` · `annual_interest_rate` 배열. 이자는 기초 잔액에 월할로 계산한다.

상환 계획이 확정되지 않았으면 창업자에게 **원리금 균등인지 만기 일시인지**를 먼저 묻는다. 둘은 현금 곡선이 완전히 다르다.

---

## equity — 증자·배당

`raises` · `dividends` 배열. 조달 라운드가 예정되어 있으면 그 달에 `raises`를 넣는다.

`statement_role: equity_raise` line item으로도 표현할 수 있다. 둘 다 쓰면 이중 계상되므로 **한쪽만 쓴다.**

---

## tax — 법인세

| `policy` | 동작 |
|---|---|
| 생략 또는 `none` | 세금을 계산하지 않는다 |
| `rate` | `rate`를 EBIT에 곱한다 |
| `direct` | `expense` 배열을 그대로 쓴다 |

`none`과 `rate`·`expense`는 함께 줄 수 없다. `rate` policy에 `expense`를, `direct` policy에 `rate`를 주면 거부된다.

세율을 모른다고 `rate: 0`으로 우회하지 않는다. 생략하면 `none`이고 경고가 나온다.

```
tax: 세금을 계산하지 않습니다(policy=none). 흑자 구간의 현금이 실제보다
낙관적으로 나옵니다
```

**적자 구간에서는 두 모드의 기말현금이 정확히 일치한다** — 법인세가 0이기 때문이다. 흑자로 돌아서는 순간부터 갈린다.

---

## solver — 최소현금·리볼버

`revolver_enabled` · `minimum_cash` · `maximum_revolver` · `initial_revolver` · `tolerance` · `max_iterations`.

현금이 `minimum_cash` 아래로 내려가면 자동으로 차입하고, 이자↔차입잔액↔현금의 순환을 반복 계산으로 푼다. 수렴하지 않으면 `solver_not_converged` 진단이 나온다.

**리볼버를 켜면 워크북을 낼 수 없다.** Excel은 반복 계산이 기본으로 꺼져 있어 같은 순환을 재현하지 못한다. 사용자가 조작 가능한 스프레드시트를 원한다면 **리볼버를 켜기 전에 그 사실을 알린다.**

---

## 불변식 — 결과에서 반드시 확인한다

accrual 결과의 `invariant_checks` 5종은 전부 `passed: true`여야 한다.

| id | 검사 |
|---|---|
| `fm.v2.invariant.balance_sheet` | 자산 = 부채 + 자본 |
| `fm.v2.invariant.cash_reconciliation` | 현금흐름표 기말현금 = BS 현금 |
| `fm.v2.invariant.retained_earnings` | 이익잉여금 롤포워드 |
| `fm.v2.invariant.ppe_rollforward` | PP&E 롤포워드 |
| `fm.v2.invariant.debt_rollforward` | 차입금 롤포워드 |

하나라도 실패하면 **그 결과를 IR이나 실사 자료에 쓰지 않는다.** 숫자를 인용하기 전에 이 다섯 줄을 먼저 본다.
