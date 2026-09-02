# 워크북 전사 — `--workbook` 봉투를 xlsx로 옮긴다

`financial-model --workbook`은 xlsx 바이너리를 만들지 않는다. **시트·행·셀별 수식 문자열**을 JSON으로 낸다. 그 JSON을 파일로 옮기는 것이 전사다.

**신뢰 모델: 설계하지 않고 전사만 한다.** 계약에 있는 것만 쓰고, 계약에 없는 것은 짓지 않는다. 수식을 읽고 다시 타이핑하지 않는다 — accrual 감가상각 수식은 36개월 실측에서 **2,358자**다. 사람도 LLM도 그것을 옮겨 적을 수 없다. 반드시 **스크립트가 JSON을 읽어 셀에 넣는다.**

## 0. 먼저 `supported`를 본다

```python
if not workbook["supported"]:
    print(workbook["unsupported_reason"])   # 리볼버가 켜져 있으면 시트가 없다
```

`sheets: []`인 워크북으로 파일을 만들지 않는다. 빈 워크북을 넘기면 받는 쪽은 재무제표가 들어 있다고 오해한다. 이유를 그대로 사용자에게 전하고 멈춘다.

## 0.5. 봉투를 먼저 파일로 만든다

전사 스크립트는 봉투 **파일**을 받는다. 36개월 accrual 봉투는 실측 **746,312자**다 —
받아 적을 수 있는 크기가 아니다.

| 어디서 부르나 | 봉투가 파일이 되는 경로 |
|---|---|
| 원격 MCP + Claude Code | 도구 결과가 한도를 넘으면 하니스가 `~/.claude/projects/<프로젝트>/<세션>/tool-results/<도구>-<ts>.txt`에 **원문 그대로** 떨군다. 그 경로를 스크립트 인자로 넘긴다 |
| 원격 MCP + 그 밖 | 호스트가 큰 결과를 파일로 주지 않으면 전사할 방법이 없다. 봉투를 요약해서 옮기지 말고 그 사실을 알린다 |

**전사만 할 것이면 `--summary`를 같이 준다.** 축약본도 워크북을 그대로 싣는다 —
줄어드는 것은 `statements`·`schedules`·`trace` 같은 표준 결과뿐이고 워크북은 손대지 않는다.
36개월 accrual 실측으로 **446 KB → 152 KB(66% 절감)** 이고, 전사·재계산 결과는 같다
(수식 셀 2,309/2,309, 불일치 0). 원격 MCP처럼 컨텍스트가 빠듯한 자리에서 특히 그렇다.

이전 엔진(플러그인 0.7.0 미만)은 이 조합에서 **워크북을 조용히 버렸다.** 축약본에 `workbook`이
없으면 엔진이 옛 버전이므로, `--summary` 없이 다시 부른다.

## 1. 환경이 가르는 것은 전달 경로뿐이다

| 환경 | 전사 | 사용자에게 넘기는 법 |
|---|---|---|
| Claude Code · Codex | openpyxl로 작업 디렉터리에 쓴다 | 파일 경로를 알린다 |
| Claude Desktop · 웹 | xlsx 스킬의 코드 실행 환경에서 **같은 openpyxl 코드**를 돌린다 | 그 환경의 파일 다운로드 |

**전사 코드는 두 환경이 같다.** 환경이 가르는 것은 파일을 어떻게 넘기느냐다. Desktop·웹에서 xlsx 스킬의 서식 절차(열 너비·머리글 스타일)를 얹는 것은 무방하다 — 표시 전용이고 수식·값을 건드리지 않는 한 계약 위반이 아니다.

## 2. 계약에서 읽는 것

| 필드 | 쓰는 곳 | 지어내면 생기는 일 |
|---|---|---|
| `layout.month_columns[i]` | `months[i]`의 열 문자 | `chr(ord("G")+i)`류는 21번째 달 `Z→AA`에서 어긋난다. 수식 안 참조는 `AA`를 가리키므로 **`#REF!`도 안 뜨고 값만 조용히 틀린다** |
| `layout.header_labels` | 헤더 행 A~F 텍스트 | 실행마다 워크북이 달라진다 |
| `layout.label_column`·`id_column`·`unit_column` | 행의 A·B·C | — |
| `months[i]` | 월 열 헤더 | — |
| `row.formulas[i]` | 월 셀 내용. `=`로 시작하면 수식, 아니면 리터럴, **빈 문자열이면 셀을 비운다** | 빈 문자열을 리터럴로 쓰면 그 셀이 문자열이 되고 그것을 읽는 수식이 깨진다 |
| `row.parameters[]` | D·E·F 가정 셀. `column`·`value`·`number_format` | 가정 셀이 없으면 사용자가 바꿀 자리가 없다 — 워크북을 내는 이유가 사라진다 |
| `row.number_format` | 그 행 월 셀의 표시 서식 | 세율 0.22가 `0.22`로 보인다 |
| `param.number_format` | 그 파라미터 셀의 표시 서식. **행 서식보다 우선한다** | `growth`의 `monthly_growth`, `cohort`의 `retention`은 행이 `count`여도 비율이다 |
| `sheets[]` 순서, `rows[].row` | 시트·행 위치 | 수식이 절대행 참조(`Assumptions!G$3`)를 쓴다. 행을 옮기면 전부 깨진다 |

**시트 순서·행 번호를 재배치하지 않는다.** 정렬·그룹핑·행 삽입 전부 금지다.

## 3. 전사 스크립트

```python
"""workbook 봉투 → xlsx. 계약이 준 문자열을 그대로 옮긴다."""
import json, sys
from openpyxl import Workbook

envelope = json.load(open(sys.argv[1]))
wb = envelope.get("data", envelope).get("workbook", envelope.get("workbook"))

if not wb.get("supported", True):
    print(f"미지원 모델: {wb.get('unsupported_reason', '')}")
    sys.exit(2)

layout, months = wb["layout"], wb["months"]
month_cols = layout["month_columns"]          # 열은 계약이 준다. 직접 계산하지 않는다.
header_row = layout["header_row"]

book = Workbook()
book.remove(book.active)
for sheet in wb["sheets"]:                    # 시트 순서 그대로
    ws = book.create_sheet(sheet["name"])
    for header in layout["header_labels"]:
        ws[f'{header["column"]}{header_row}'] = header["label"]
    for col, month in zip(month_cols, months):
        ws[f"{col}{header_row}"] = month
    for row in sheet["rows"]:                 # 행 순서 그대로
        r = row["row"]
        ws[f'{layout["label_column"]}{r}'] = row["label"]
        ws[f'{layout["id_column"]}{r}'] = row["id"]
        ws[f'{layout["unit_column"]}{r}'] = row["unit"]
        for param in row.get("parameters") or []:
            cell = ws[f'{param["column"]}{r}']
            cell.value = param["value"]
            cell.number_format = param["number_format"]
        for col, formula in zip(month_cols, row["formulas"]):
            if formula == "":                 # 빈 문자열은 셀을 비워 둔다
                continue
            cell = ws[f"{col}{r}"]
            cell.value = formula if formula.startswith("=") else float(formula)
            cell.number_format = row["number_format"]
    # 표시 전용. 값·수식에 영향 없음.
    ws.freeze_panes = f"{month_cols[0]}{header_row + 1}"
    ws.column_dimensions[layout["label_column"]].width = 34
    ws.column_dimensions[layout["id_column"]].width = 28
book.save(sys.argv[2])
```

`values`는 시트에 쓰지 않는다. 값을 써 넣으면 가정을 바꿔도 아무것도 재계산되지 않아 워크북이 아니라 표가 된다.

## 4. 자체 검증 — 파일을 넘기기 전에 돌린다

전사는 조용히 실패한다. 열이 하나 밀려도 `#REF!`가 뜨지 않고 값만 틀리므로, **반드시 다시 열어 대조한다.**

검증은 두 단계다.

| 단계 | 무엇을 보나 | 필요한 것 |
|---|---|---|
| 1. 전사 충실도 | 수식 문자열 원문 일치, 수식 셀 수 = JSON 수식 수, 빈 셀, 리터럴, 파라미터 값·서식, 시트 순서 | openpyxl만 |
| 2. 재계산 대조 | 캐시값 = 봉투 `values` | LibreOffice/Excel로 한 번 열어 저장해야 캐시값이 생긴다 |

2단계는 LibreOffice headless로 5초면 된다. openpyxl이 쓴 파일에는 캐시값이 없으므로
LibreOffice가 전부 계산해서 저장한다.

```bash
soffice --headless --norestore -env:UserInstallation=file:///tmp/lo-profile \
        --convert-to xlsx --outdir recalc/ <파일>.xlsx
```

`--outdir`를 원본과 다른 곳으로 준다. 검증 스크립트에는 **재계산본** 경로를 넘긴다.

**openpyxl은 수식을 계산하지 않는다.** 방금 쓴 파일을 `data_only=True`로 읽으면 수식 셀은 전부 `None`이다 — 이것은 오류가 아니다. 그 자리에서 할 수 있는 것이 1단계이고, 1단계만으로도 열 어긋남·빈 셀 오기·서식 누락은 전부 잡힌다.

```python
"""전사한 xlsx를 다시 열어 봉투와 대조한다."""
import datetime, json, sys
from openpyxl import load_workbook


def as_number(value):
    """날짜 서식(yyyy-mm) 셀은 openpyxl이 읽을 때 datetime으로 되돌린다.
    파일은 멀쩡한데 대조만 깨지므로 Excel serial 숫자로 되돌린다."""
    if isinstance(value, datetime.datetime):
        return (value - datetime.datetime(1899, 12, 30)).total_seconds() / 86400
    if isinstance(value, datetime.date):
        return (value - datetime.date(1899, 12, 30)).days
    if isinstance(value, datetime.time):
        return (value.hour * 3600 + value.minute * 60 + value.second) / 86400
    if isinstance(value, datetime.timedelta):
        return value.total_seconds() / 86400
    return value


def same_format(want, got):
    """LibreOffice로 재계산해 저장하면 서식의 `-`가 `\\-`로 이스케이프된다
    (`yyyy-mm` → `yyyy\\-mm`). 표시 결과는 같으므로 백슬래시를 지우고 본다.
    `#,##0` vs `0.00%` 같은 진짜 차이는 그대로 남는다."""
    return (want or "").replace("\\", "") == (got or "").replace("\\", "")


envelope = json.load(open(sys.argv[1]))
wb = envelope.get("data", envelope).get("workbook", envelope.get("workbook"))
path = sys.argv[2]
layout, months, cols = wb["layout"], wb["months"], wb["layout"]["month_columns"]
book = load_workbook(path, data_only=False)
cached = load_workbook(path, data_only=True)

bad, formula_cells, literal_cells, blank_cells = [], 0, 0, 0
expected_formulas, value_seen = 0, 0

want = [s["name"] for s in wb["sheets"]]
if book.sheetnames != want:
    bad.append(("<book>", "sheet_order", want, book.sheetnames))

for sheet in wb["sheets"]:
    ws, wsc = book[sheet["name"]], cached[sheet["name"]]
    for header in layout["header_labels"]:
        ref = f'{header["column"]}{layout["header_row"]}'
        if ws[ref].value != header["label"]:
            bad.append((sheet["name"], ref, header["label"], ws[ref].value))
    for offset, month in enumerate(months):
        ref = f'{cols[offset]}{layout["header_row"]}'
        if ws[ref].value != month:
            bad.append((sheet["name"], ref, month, ws[ref].value))
    for row in sheet["rows"]:
        r = row["row"]
        for column, expected in ((layout["label_column"], row["label"]),
                                 (layout["id_column"], row["id"]),
                                 (layout["unit_column"], row["unit"])):
            if ws[f"{column}{r}"].value != expected:
                bad.append((sheet["name"], f"{column}{r}", expected, ws[f"{column}{r}"].value))
        for param in row.get("parameters") or []:
            cell = ws[f'{param["column"]}{r}']
            if as_number(cell.value) != param["value"]:
                bad.append((sheet["name"], f'{param["column"]}{r}', param["value"], cell.value))
            if not same_format(param["number_format"], cell.number_format):
                bad.append((sheet["name"], f'{param["column"]}{r} 서식',
                            param["number_format"], cell.number_format))
        for offset, formula in enumerate(row["formulas"]):
            ref = f"{cols[offset]}{r}"
            got = as_number(ws[ref].value)
            if formula == "":                       # 빈 셀이어야 한다. values는 대조 대상이 아니다
                blank_cells += 1
                if got is not None:
                    bad.append((sheet["name"], ref, "<빈 셀>", got))
                continue
            if formula.startswith("="):
                expected_formulas += 1
                if isinstance(got, str) and got.startswith("="):
                    formula_cells += 1
                if got != formula:                  # 수식은 원문 그대로여야 한다
                    bad.append((sheet["name"], ref, formula[:60], str(got)[:60]))
            else:
                literal_cells += 1
                if got is None or abs(float(got) - float(formula)) > 1e-9:
                    bad.append((sheet["name"], ref, formula, got))
            if not same_format(row["number_format"], ws[ref].number_format):
                bad.append((sheet["name"], f"{ref} 서식", row["number_format"], ws[ref].number_format))
            if not formula.startswith("="):
                continue
            got_value = as_number(wsc[ref].value)   # 2단계. 재계산해 저장한 파일에만 있다
            if isinstance(got_value, (int, float)):
                value_seen += 1
                if abs(float(got_value) - row["values"][offset]) > 0.05:
                    bad.append((sheet["name"], f"{ref} 값", row["values"][offset], got_value))

print(f"[{wb.get('mode', 'cash')}] 수식 셀 {formula_cells}/{expected_formulas} · "
      f"리터럴 {literal_cells} · 빈 셀 {blank_cells} · 불일치 {len(bad)}")
print(f"재계산 대조: 수식 셀 {value_seen}/{expected_formulas}" if value_seen
      else "재계산 대조: 캐시값 없음 — 전사 충실도만 확인했다.")
for row in bad[:12]:
    print("  MISMATCH", row)
sys.exit(1 if bad else 0)
```

### 대조에서 빼는 것 세 가지

- **수식이 빈 문자열인 셀.** 기초잔액 11행 + 내용연수 + 세율 = 13행의 월 셀이 전부 빈 문자열인데 그 행의 `values`는 0이다. 그대로 대조하면 36개월 기준 **468건이 전부 오탐**으로 뜬다. 그 행의 값은 D열 파라미터 셀에 있고, 파라미터는 따로 대조한다.
- **날짜 서식 셀의 타입.** `value_type: "date"` 행은 `number_format: "yyyy-mm"`인데, openpyxl은 그런 셀을 읽을 때 숫자를 `datetime`으로 되돌린다. 파일은 멀쩡한데 대조만 깨진다 — 실측에서 `Assumptions!D3`의 0이 `datetime.time(0, 0)`으로 돌아왔다. `as_number()`로 정규화한다.
- **재계산본의 서식 이스케이프.** LibreOffice로 재계산해 저장한 파일은 서식의 `-`를 `\-`로 이스케이프한다(`yyyy-mm` → `yyyy\-mm`). 표시 결과는 같은데 문자열 비교만 깨진다 — 실측에서 60개월 cash 모델의 date 행 때문에 **61건이 전부 오탐**이었다(값·수식 불일치는 0). `same_format()`으로 백슬래시를 지우고 본다. 이 정규화는 `#,##0` vs `0.00%` 같은 진짜 차이는 그대로 잡는다.

허용오차는 **0.05원**이다. 엔진의 money 반올림 단위는 0.01원이고, 수식은 엔진 반올림을 `ROUND`로 재현하고 있다. 남는 차이는 재계산 엔진의 부동소수 누적에서 나온다.

### 불일치가 나오면

**파일을 제공하기 전에 보고한다.** 불일치를 안고 넘기지 않는다. 시트·셀 참조·기대값·실제값을 그대로 적고, 원인을 짐작해 수식을 고쳐 쓰지 않는다 — 수식을 손보는 순간 전사가 아니라 설계가 된다.

## 5. 파일명과 해시

- 파일명: `<회사>-<모델>-<YYYYMMDD>.xlsx` (예: `acme-base-accrual-20260824.xlsx`). `<모델>`은 시나리오·모드 식별자다.
- 계산 해시(`data.calculation_hash`)는 **자르지 말고 한 번만** 인용한다. 워크북은 해시에 들어가지 않으므로(`--workbook` 여부가 해시를 바꾸지 않는다) 해시는 이 워크북이 옮긴 계산을 가리킨다.
- **워크북 안의 수식을 채팅에서 설명하거나 요약하지 않는다.** 설명은 워크북 안에 있다(A열 label, B열 id). 숫자를 다시 적는 순간 원칙 1을 어긴다.

## 실측 (2026-08-24, ORBIT #4871)

이 문서의 두 스크립트를 그대로 돌린 결과다.

| 봉투 | 모드 | 수식 셀 | 리터럴 | 빈 셀 | 1단계 불일치 | 재계산 대조 |
|---|---|---|---|---|---|---|
| `accrual/workbook-36.input.json` | accrual | 2,309 | 391 | 468 | 0 | 2,700셀 0 불일치 |
| `legacy/v1/boundary-60.input.json` (60개월, `Z` 넘김) | cash | 600 | 120 | 60 | 0 | 720셀 0 불일치 |

재계산 대조는 Python `formulas` 라이브러리로 돌렸다. 계약은 이 문서의 전사 절차와 검증 게이트를
정본으로 삼으며, 수식 셀 수·수식 원문·빈 셀·파라미터·서식이 하나라도 다르면 파일을 제공하지 않는다.

**0 불일치가 검사를 안 한 결과는 아니다.** 전사기를 일부러 망가뜨려 검증이 잡는지 확인했다.

| 망가뜨린 것 | 검증 결과 |
|---|---|
| `chr(ord("G")+i)`로 열을 계산 (21번째 달부터 `AA` 대신 `GA`) | 수식 셀 1,285/2,309, **불일치 2,496** — 첫 보고가 `AA1` |
| 월 열을 한 칸 밀어 씀 | **불일치 617** |
| 수식 대신 `values` 캐시값을 써 넣음 | 수식 셀 **0/2,309**, 불일치 600 |

첫 줄이 이 문서가 열을 계산하지 말라고 하는 이유다. 60개월에서는 그 구현이 유효하지 않은 열 문자를 만들어 저장 단계에서 바로 죽지만, **36개월에서는 파일이 멀쩡히 저장된다** — 검증하지 않으면 그대로 나간다.

## 실측 2 (2026-08-24, ORBIT #4872) — LibreOffice 재계산

절차서를 처음 읽는 별도 `claude -p` 세션에 스펙 없이 회사 상황만 주고 36개월 accrual 워크북을 시켰다.
그 세션은 3절 스크립트를 **바이트 단위로 그대로** 썼고, 4절 검증을 돌린 뒤 그 숫자를 사용자에게 보고했다.

| 봉투 | 수식 셀 | 리터럴 | 빈 셀 | 1단계 불일치 | 재계산 오차 0 | 최대 \|오차\| |
|---|---|---|---|---|---|---|
| 아크메랩스 36개월 (자식 세션) | 2,486 | 106 | 468 | 0 | 2,486/2,486 | 0 |
| 아크메랩스 36개월 (재현) | 2,522 | 70 | 468 | 0 | 2,522/2,522 | 0 |
| `accrual/workbook-36.input.json` | 2,309 | 391 | 468 | 0 | 2,068/2,309 | 0.01원 |

LibreOffice 26.2.5.2 headless. `formulas` 라이브러리 대신 실제 스프레드시트 엔진으로 다시 잰 것이다.
운전자본이 매월 움직이는 `workbook-36` 픽스처에서만 241개 셀이 0.005~0.01원 벌어진다(허용오차 0.05원 안).
운전자본이 0인 모델에서는 오차가 아예 없다.

**배포된 원격 MCP에는 이 절차가 아직 통하지 않는다.** 워커가 `layout.month_columns`·`header_labels`·
`number_format`이 없는 이전 계약을 내고, `engine_version` 문자열은 둘을 구분하지 못한다.
`KeyError: 'month_columns'`가 나면 **열을 직접 계산해 메우지 말고 멈춘다**. 필요한 원격 계약 필드는
`layout.month_columns`·`header_labels`·`number_format`이며, 하나라도 없으면 검증 가능한 워크북을 만들 수 없다.
