# fintool-plugin

**랜딩 페이지(플러그인 유무 Before/After 대조): https://risingdream.github.io/fintool-plugin/**

스타트업 재무 계산 도구(원격 MCP)와 재무·IR 스킬 10종을 묶은 에이전트 플러그인 마켓플레이스다.
Claude Desktop·Claude Code·Codex가 이 주소 하나로 같은 플러그인을 설치하고 갱신한다.

계산의 정본은 원격 MCP `https://fintool-mcp.risingdream.workers.dev/mcp`
(`fintool_catalog`·`fintool_run` 두 도구)이고, 이 저장소는 스킬과 MCP 설정을 담는다.

### 공개 범위

| 층 | 라이선스 | 공개 |
|----|---------|------|
| 이 저장소(스킬·MCP 설정) | MIT | 공개 |
| 계산 엔진 소스 | — | 비공개 |

계산 엔진은 공개하지 않는다. 대신 결과 봉투에 `schema_version`·`engine_version`·`calculation_hash`가 실려,
**같은 입력을 같은 엔진 버전으로 다시 돌리면 같은 해시가 나오는지**를 누구나 대조할 수 있다.

## 플러그인 설치

| 클라이언트 | 방법 |
|------------|------|
| Claude Desktop | 설정 → 플러그인 → 추가 → **저장소에서 추가** → `risingdream/fintool-plugin` |
| Claude Code | `/plugin marketplace add risingdream/fintool-plugin` 후 `/plugin install fintool@fintool-plugin` |
| Codex | `codex plugin marketplace add risingdream/fintool-plugin` |

첫 도구 호출 때 `fintool` 커넥터의 OAuth 동의 화면이 한 번 뜬다. 허용하면 그 뒤로는 묻지 않는다.

## 들어 있는 것

```text
plugins/fintool/
  .claude-plugin/plugin.json   name fintool, semver
  .codex-plugin/plugin.json    Codex용 동일 메타
  .mcp.json                    fintool → 원격 MCP URL
  skills/                      재무 6: startup-finance · valuation · financial-statements
                                      fund-performance · portfolio-risk · fixed-income
                               검증 4: startup-design · startup-competitors
                                      startup-positioning · startup-pitch
  desktop/                     Claude Desktop 수동 커넥터 스니펫
```

## 갱신

| 층 | 어떻게 |
|----|--------|
| 계산 도구 | 서버 재배포. 플러그인 재설치 없음 |
| 스킬 | `plugin.json`의 `version`이 올라가면 마켓플레이스 갱신(`/plugin marketplace update`, `codex plugin marketplace upgrade`, Desktop 플러그인 업데이트)으로 받는다 |

`plugins/fintool/`은 비공개 엔진 저장소에서 GitHub Actions가 동기화한다.
여기에 직접 낸 PR은 다음 동기화 때 덮어쓰이므로, 스킬 수정 제안·버그 신고는
[Issues](https://github.com/risingdream/fintool-plugin/issues)로 보내면 원본에 반영해 내려보낸다.

## 라이선스

이 저장소의 내용(스킬 문서·MCP 설정)은 MIT다. 계산 엔진 소스는 포함되지 않으며 비공개다.
