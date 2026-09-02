---
name: distress-restructuring
description: 법인회생, 법인파산, 도산, 청산가치, 계속기업가치, 회생계획, 채권 회수율 요청에서 외부 확정 법률 분류와 최신 근거를 받은 뒤 fintool MCP의 restructuring으로 청산·회생 경제성을 계산하고 merger로 M&A 대안을 분리 비교한다. 법률자문, 채권 분류 자동판정, 신청 적격성·인가 가능성 예측은 하지 않는다.
---

# Distress Restructuring

## 역할과 경계

이 스킬은 법인 도산의 경제성 분석만 수행한다. 법률가·관재인·법원 문서가 확정한 채권 분류,
담보 배분, 우선순위, 회생계획 조건과 사용자가 승인한 가치평가 가정을 입력받아 청산 배당,
회생 회수 NPV, 청산가치보장 산술, 계속기업가치와 현금 이행 가능성을 설명한다.

계산기는 법률 규칙을 선택하지 않는다. 법률 근거와 사실관계가 숫자 입력보다 먼저다.

## 원격 MCP 호출 계약

첫 계산 전에 `fintool_catalog {"tool":"restructuring"}`로 최신 flags,
`input_contract.call_example`, typed workflow ports를 확인한다. 계산은
`fintool_run {"tool":"restructuring","flags":{"mode":"compare","input":{"schema_version":"restructuring-compare/v1","liquidation":{},"rehabilitation":{}}}}`
형태로 실행하되 빈 객체를 실제 strict request로 교체한다.

`flags`는 JSON 객체다. `flags.spec`도 JSON 객체이며 JSON 문자열로 이중 직렬화하지 않는다.
`restructuring`은 현재 `spec`이 아니라 `input` 객체를 사용한다. 원격 MCP는 호출자 로컬의
`@파일`을 읽지 못하므로 catalog 예제의 객체형 `input`을 직접 전달한다.

## 고정 실행 순서

1. **사실 수집** — 법인·사건·기준일·통화·금액 단위, 채권과 담보, 계획·가치평가·현금 자료를 모으고 missing과 명시적 0을 구분한다.
2. **법률 분류·근거 확인** — 외부 확정 분류를 source에 연결하고 사건 시점의 적용 법령과 인용 실존을 확인한다.
3. **FINTOOL 계산** — 의도에 맞는 `liquidation`·`rehabilitation`·`compare`를 완성된 입력마다 한 번 실행하고 성공 봉투만 사용한다.
4. **보존식 검산** — 청산의 `conservation`과 각 mode의 `invariant_checks`를 실제 출력 경로에서 확인하고 실패가 남으면 결과를 승인하지 않는다.
5. **한계·다음 행동** — 확인된 사실과 가정, 계산 결과, 미확인 법률 쟁점, 필요한 전문가 검토를 분리해 제시한다.

순서를 건너뛰지 않는다. 특히 숫자가 충분해 보여도 법률 분류·근거 확인 전에는 FINTOOL을
호출하지 않는다.

## 사실 수집

다음 입력을 같은 통화·단위와 명시된 기준일로 받는다.

- 분석 목적: 법인 청산, 회생계획, 두 대안 비교, 또는 M&A까지 포함한 대안 검토
- 법인과 사건 식별자, 분석 `as_of`, 법률 적용을 확인할 신청·발생·평가 시점
- 채권자·claim component별 인정액, 외부 확정 `legal_class`, `waterfall_rank`, 지급시점
- 담보 목적물별 총환가액·직접비용·외부 확정 claim/재단 배분
- 회생 claim별 현금 일정, 비현금 face amount와 외부 공정가치, 명시적 면제
- claim별 청산 비교액, 청산 자산가치·비용, 계속기업 현금흐름·terminal value·비영업자산
- 기간별 opening cash·cash sources·mandatory outflows와 각 입력의 source
- 할인율·반올림 규칙·허용오차의 승인 근거

같은 채권의 담보부와 부족액은 외부 검토가 분리한 component 그대로 받는다. 금액 0은 유효한
명시 입력이지만 누락은 0이 아니다. 업계 관행, 이름이 비슷한 채권, 목표 회수율에서 값을
역산하지 않는다.

## 법률 근거와 provenance

변동 법령은 스킬이나 도구에 하드코딩하지 않는다. 법률 근거가 필요한 실행마다 다음을 수행한다.

1. 사건 기준일에 `korean-law`의 `applicable_law`를 호출해 시행판과 경과규정을 확인한다.
2. 답변·법률검토서가 인용한 조문은 `verify_citations`로 실존을 확인한다.
3. 결과를 법령 source의 `publisher`, `title`, `as_of`, `url`, `law_mst`,
   `effective_from`, `verification_status`에 기록한다.
4. `korean-law`을 쓸 수 없으면 국가법령정보센터·법원·공시기관의 공식 원문처럼 검증 가능한
   외부 근거만 사용하고 발행자·제목·URL·시행일·접근일을 남긴다.

검색 실패, 부분 검증, 시행시점 불일치, 공식 원문 미확보를 “확인됨”으로 바꾸지 않는다.
법령 source의 `as_of`와 사실·평가 source의 `as_of`를 섞지 않는다. 법적 분류 자체는
담당 법률가·관재인·법원 문서의 source가 확정해야 하며, 법령 조문이 존재한다는 사실만으로
개별 claim의 분류가 확정되지 않는다.

## 중단 게이트

다음 중 하나면 계산을 중단한다.

- 누락 입력이 남아 strict request나 보존식을 완성할 수 없다.
- 법적 분류 불명확, 담보 배분·우선순위 미확정, 회생계획 조건 충돌이 남아 있다.
- 최신 근거 부재, `applicable_law` 실패, 인용 확인필요 등으로 적용 법령을 확인하지 못했다.
- 통화·단위·기준일·채권 범위가 대안 사이에 맞지 않는다.
- source가 입력값을 지지하지 않거나 같은 가치가 여러 대안에 중복 포함됐는지 확인할 수 없다.

중단 응답은 다음 구조로 제시한다.

- `status: not_calculated`
- 확인되지 않은 항목과 계산에 미치는 영향
- 필요한 근거·필드·확정 주체
- 안전한 다음 행동과 전문가에게 확인할 질문
- 현재 한계와 사용하지 않은 가정

빠진 값을 0으로 채우지 않는다. 법률 근거가 없다는 이유로 가장 흔한 분류를 택하거나 숫자만
먼저 계산하지 않는다.

## FINTOOL 계산

mode는 질문과 완성된 입력 범위로 고른다.

| 요청 | route |
| --- | --- |
| 확정 담보 배분과 파산 워터폴 | `restructuring` + `mode=liquidation` |
| 확정 회생계획의 회수·가치·현금 이행 | `restructuring` + `mode=rehabilitation` |
| 같은 통화·기준일의 청산과 회생 | `restructuring` + `mode=compare` |

catalog의 strict 예제를 복사해 `source_records`와 모든 source reference를 채운다.
성공 봉투의 `data`만 사용하며 실패 봉투·부분 입력에서 회수율을 만들지 않는다.
`warnings`, `source_records`, `invariant_checks`와 mode별 구조화 결과를 숫자와 함께 보존한다.
다른 도구를 조합할 때도 그 도구가 실제 반환한 필드만 인용한다.

## 보존식 검산

숫자를 설명하기 전에 다음을 확인한다.

### 청산

- 담보 총환가액 = 직접비용 + claim/재단 배분 + 재단 잔여 유입
- 재단 pool = 비담보 claim 배당 + `ending_estate_cash`
- `conservation.balanced`가 true이고 `conservation.difference`가 계약 허용오차 안이다.
- 모든 `invariant_checks[].passed`가 true이며 `invariant_checks[].actual`과
  `invariant_checks[].expected`가 일치한다.
- claim·creditor·class 합계가 totals와 일치하고 선순위 미변제 중 후순위 배당이 없다.

### 회생

- `claim_results`에서 인정액 = 현금 face + 비현금 face + 명시적 waiver인지 확인한다.
- `claim_results[].economic_recovery_npv` = 현금 NPV + 외부 공정가치 NPV인지 확인한다.
- claim별 `liquidation_comparator_npv`와 회생 회수 NPV의 차이가 청산가치보장 산술과 일치한다.
- `cash_feasibility.periods`의 현금 roll-forward와 `funding_gap`이 보존되고 자동 차입이 숨어 있지 않다.
- `valuation`, `claim_results`, `cash_feasibility.periods`를 근거로 모든
  `invariant_checks[].passed`가 true이며 `actual`과 `expected`가 일치하는지 확인한다.

### 비교

- 두 request의 통화와 `as_of`가 같고 creditor 연결 범위가 명시돼 있다.
- 중첩 `liquidation`·`rehabilitation` 원결과를 그대로 보존하며 차이만 `comparison`에서 읽는다.

필수 보존식이 깨졌거나 차이가 허용오차를 넘으면 결과를 승인하지 않는다. 성공한 봉투의
숫자를 재계산하지 않는다. 반올림된 표시값으로 새 합계를 만들지 않는다.

## 대안 라우팅

- 청산과 회생은 `restructuring compare`가 한 번 계산한 원결과와 차이를 사용한다.
- M&A 대안은 `fintool_catalog {"tool":"merger"}` 뒤 `merger`가 계산한 EV bridge,
  consideration, funding gap, seller proceeds 봉투를 사용한다.
- 가치·가격·현금 가정을 흔들 때는 `scenario`, 승인된 typed 연결은 `workflow`를 사용한다.
- 동일 통화, 동일 기준일, 동일 범위가 확인된 경우에만 청산·회생·M&A를 나란히 제시한다.
- 비교표는 각 도구의 성공 봉투와 실제 반환된 식별·근거 필드만 가리키며 중복 계산하지 않는다.

`restructuring`의 계속기업가치를 `merger`의 enterprise value로 자동 승격하지 않는다.
M&A 성사 가능성·절차·세금과 회생계획의 법률 효과는 별도 근거다. 대안 간 범위가 다르면
억지로 단일 순위를 만들지 말고 차이와 필요한 정규화 입력을 설명한다.

## 결과 전달

결론보다 먼저 계산 가능 여부를 밝히고 다음을 구분한다.

- 기준일·통화·단위와 법률·사실·가정 source
- mode와 case ID, 총 회수 NPV, creditor별 회수율, 가치 차이와 funding gap
- 청산의 `conservation.balanced`·`conservation.difference`와 mode별 `invariant_checks`
- 비현금 회수의 face amount와 외부 공정가치
- 경계 warning, 미확인 항목, 결과가 답하지 않는 법률·실행 질문
- 전문가 검토 또는 추가 자료 수집의 다음 행동

`PASS`, `FAIL`, 높은 회수율, 양의 계속기업가치 차이는 산술 결과다. 인가·동의·집행·거래
성공의 결론으로 표현하지 않는다.

## 금지와 한계

- 법률자문, 회생·파산 신청서나 회생계획안 작성
- 채권 분류 자동판정, 담보 효력·별제권·우선순위·조 편성·형평성 판단
- 신청 적격성, 관할, 개시·인가 가능성, 배당·소송·법원 판단을 예측하지 않는다.
- 개인회생·개인파산·면책 요청을 법인 모델로 계산
- 청산 할인율, 회수율, 현금흐름, terminal value, 비현금 공정가치의 숨은 추정
- 법령 검색 실패나 오래된 source를 최신·검증 완료로 표시
- FINTOOL 봉투 밖에서 배당, NPV, 회수율, 가치 차이, funding gap을 다시 계산
