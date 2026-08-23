# Before / After 트랜스크립트 원문

랜딩 페이지 Before/After 열의 출처. 2026-08-23 Claude Code `claude -p`(비대화형)로 같은 프롬프트를 플러그인 off / on 상태에서 실제 실행한 기록이다. 응답 본문·도구 호출·도구 결과를 가공하지 않았다(홈 디렉터리 경로만 `~`로 치환).

| 파일 | 조건 |
|------|------|
| `case-*-off.txt` | `claude plugin disable fintool@fintool-plugin`, `claude -p --tools "" --setting-sources ""` — 셸·파일 도구 없음, 사용자 CLAUDE.md 없음. 모델 단독(Claude Desktop의 일반 채팅과 같은 조건) |
| `case-*-on.txt` | `claude plugin enable fintool@fintool-plugin`(A는 v0.5.2, C는 v0.5.3, B는 v0.5.4 — `convertible` TF 커맨드 추가 후), `claude -p --tools "Skill,Read" --mcp-config <plugin .mcp.json> --strict-mcp-config`, 실행 동안 `~/.claude/CLAUDE.md` 제거 — 스킬 + 원격 fintool MCP. `Read`는 Claude Code가 큰 MCP 결과를 파일로 저장하는 동작 때문에 허용(이번 실행에서는 쓰이지 않음) |

- 실행 위치: 빈 임시 폴더(`mktemp -d`). 프로젝트 파일·CLAUDE.md 힌트 없음.
- 모델: 두 조건 모두 같은 모델(파일 머리말에 기록).
- 프롬프트: A `월 매출 3천만 원, 월 30% 성장, 고정비 8천만 원, 현금 6억. 런웨이와 다음 라운드 필요액은?` / B `액면 100억 원, 표면금리 연 1%(연 1회 이표), 만기 3년(2026-08-23 발행, 2029-08-23 만기), 전환가 20,000원, 현재 주가 18,000원, 주가 변동성 35%, 무위험금리 3.2%, 이 회사 일반채 신용스프레드 4%. 이 전환사채 가치는?` / C `내년부터 5년 FCF 10억, 12억, 14억, 16억, 18억, 순부채 20억, 발행주식 100만 주. DCF로 주당가치는?`
- 형식: `[assistant]` 응답 텍스트, `[tool_use <이름>]` 호출 인자(JSON), `[tool_result]` 도구 반환 원문, `[result]` 소요 시간·턴 수·비용(USD)·입력 토큰(캐시 읽기 포함)·출력 토큰.
