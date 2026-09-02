---
name: merger-economics
description: 회사 매각·기업 인수·M&A의 EV→equity purchase price bridge, 순부채·debt-like·정상 NWC 조정, 거래대가·sources & uses, seller proceeds, rollover·earnout, PPA·goodwill, pro forma ownership, EPS accretion/dilution을 fintool MCP로 계산한다. 독립 기업가치는 valuation, LBO 수익률은 private-markets, 증권별 희석은 startup-finance의 cap-table-simulate, 다기간 재무제표는 financial-model로 보낸다. 세무·법률·회계 분류 판단은 하지 않는다.
---

# Merger Economics

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"merger"}`로 최신 플래그와 typed workflow port를 확인한다.
계산은 `fintool_run {"tool":"merger","flags":{"enterprise-value":1000,"target-cash":0,"target-debt":0,"debt-like-items":0,"target-nwc":0,"closing-nwc":0,"other-equity-adjustments":0,"stock-consideration":0,"seller-rollover":0,"earnout-fair-value":0,"buyer-cash":1000,"new-debt":0,"other-funding":0,"transaction-fees":0,"financing-fees":0,"upfront-integration-costs":0,"seller-fees":0}}` 형태로 실행한다.
`flags`는 JSON 객체다. `flags.spec`도 JSON 객체이며 JSON 문자열로 이중 직렬화하지 않는다.
이 도구에는 현재 `spec` 플래그가 없으므로 catalog가 공개한 kebab-case 숫자 플래그만 전달한다.

## 먼저 구분할 질문

- standalone enterprise/equity value 자체를 산정하는가: `valuation`의 `wacc`·`dcf`·`equity`
- 이미 정한 EV를 closing equity price와 대가·조달·seller proceeds로 잇는가: `merger`
- sponsor LBO의 debt schedule·IRR·MOIC인가: `private-markets --mode buyout`
- SAFE·전환사채·옵션풀의 증권별 희석인가: `startup-finance`의 `cap-table-simulate`
- 여러 기간의 3표·현금·revolver인가: `financial-model`
- 가격·synergy·조달 가정의 case 생성과 연결인가: `scenario`와 `workflow`

## 입력 원칙

모든 금액은 같은 통화·단위·기준일이어야 한다. 빠진 값을 관행이나 기억으로 채우지 않는다.
핵심 거래계약 17개 플래그는 0이어도 모두 보낸다. PPA 플래그 3개 또는 accretion 플래그
11개 중 하나를 보내면 그 묶음 전체를 보내 누락과 명시적 0을 구분한다.

1. headline `enterprise-value`, target cash/debt, debt-like items
2. normalized `target-nwc`, `closing-nwc`, 서명 있는 기타 equity adjustment
3. stock consideration, seller rollover, earnout acquisition-date fair value
4. buyer cash, new debt, other funding, transaction·financing·upfront integration fees
5. seller fees와, ownership 계산이 필요하면 pre-deal buyer equity
6. PPA가 필요하면 identifiable net assets·NCI·previously held interest의 fair value
7. accretion이 필요하면 buyer/target net income, buyer shares/price, 이자율·세율,
   run-rate synergy·해당 기간 realization, integration cost·intangible amortization

`earnout-fair-value`, identifiable asset fair value, debt-like 분류, NWC 정의를 모델이 판단하지 않는다.
계약서·closing statement·회계 검토에서 확정한 값을 받는다.

## 계산 흐름

1. `data.bridge.equity_purchase_price`에서 EV, net debt, debt-like, NWC adjustment를 검산한다.
2. `data.consideration`에서 cash to sellers가 stock·rollover·earnout을 차감한 residual이고
   `difference = 0`인지 확인한다.
3. `data.funding.gap`을 확인한다. 양수는 추가 cash funding 필요액, 음수는 초과 funding이다.
4. `data.seller_proceeds`에서 현금, retained/deferred value, seller fee 차감을 분리한다.
5. 선택 입력이 있을 때만 `data.ppa`, `data.ownership`, `data.accretion`을 읽는다.
6. `warnings`, `explanations`, `calculation_hash`를 함께 보존한다.

숫자를 재계산하지 않고 성공한 fintool 봉투의 값만 인용한다. null을 0으로 바꾸지 않으며,
`explanations`의 생략 사유를 함께 전달한다. EPS accretion/dilution은 가치창출의 증거가 아니므로
standalone 가치와 synergy 가치 판단을 대신하지 않는다.

## 시나리오와 typed handoff

다기간 synergy phase-in을 도구 안에서 추정하지 않는다. 기간별 realization·비용을 명시적 case로
만들어 `scenario`로 전개하고, `workflow`의 typed `$ref`로 `merger`를 반복 실행한다.
`dcf`의 `data.enterprise_value`는 `merger`의 `enterprise-value` money port로 넘길 수 있다.
각 기간을 연결할 때 buyer/target net income과 shares의 기준 기간을 섞지 않는다.

## 결과 전달

- headline EV와 equity purchase price, 순부채·debt-like·NWC·기타 adjustment
- consideration 구성과 `difference`
- cash sources, cash uses, funding gap
- seller cash at close, retained/deferred value, seller fees, net total value
- 선택 PPA의 goodwill 또는 bargain purchase와 사용자가 준 fair value 입력
- 선택 pro forma ownership과 EPS accretion/dilution, break-even run-rate synergy
- assumptions, warnings, explanations, `calculation_hash`
- 세무·법률·회계 분류 판단이 계산 밖이라는 한계

## 금지

- 세금·세후 seller proceeds, 법률상 채무 승계, 계약 문구를 계산 결과로 단정
- debt-like item·NWC·PPA asset의 회계 분류나 fair value를 창작
- funding gap을 조용히 buyer cash나 new debt로 plug
- accretive 거래를 자동으로 좋은 거래라고 추천
- 봉투 밖에서 goodwill, EPS, ownership, break-even synergy를 다시 계산
