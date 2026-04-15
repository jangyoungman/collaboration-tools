# Claude Code MCP 설치 및 설정 가이드

Claude Code(CLI)에 업무용 MCP(Model Context Protocol) 서버를 연동하여 Confluence/Jira/GitLab/Slack/Microsoft To Do 등 협업 도구를 AI 어시스턴트와 통합한 기록입니다.

마지막 업데이트: 2026-04-15

## 환경

- OS: Ubuntu 25.04
- Claude Code 바이너리: `/home/gellotin/.nvm/versions/node/v24.14.1/bin/claude` (nvm)
- 설정 파일: `~/.claude.json` (`mcpServers` 키)
- 설치 범위: `--scope user` (전역, 모든 프로젝트에 적용)

## 설치된 MCP 서버 (8개)

| MCP | 방식 | 용도 | 비고 |
|---|---|---|---|
| Google Drive (claude.ai) | HTTP | Google Drive 연동 | 기본 탑재 |
| Playwright | npx | 브라우저 자동화 | 기본 탑재 |
| asset-management | HTTP (`https://asset.indg.co.kr/mcp`) | 사내 자산관리 | - |
| microsoft-todo | node (`mcp-todo/src/index.js`) | Microsoft To Do 연동 | [jangyoungman/mcp-todo](https://github.com/jangyoungman/mcp-todo) |
| atlassian | HTTP (`https://mcp.atlassian.com/v1/mcp`) | Confluence/Jira | 공식 Rovo MCP, OAuth 인증 |
| gitlab | npx `@modelcontextprotocol/server-gitlab` | self-hosted GitLab | `gitlab.indg.co.kr` |
| fetch | uvx `mcp-server-fetch` | 웹 fetch (WebFetch 강화) | Python 기반 |
| slack | npx `@modelcontextprotocol/server-slack` | Slack 메시지 발송 | 워크스페이스: INNODIGM |

## 설치 명령어 예시

```bash
# HTTP 기반 MCP
claude mcp add --scope user --transport http atlassian https://mcp.atlassian.com/v1/mcp

# npx 기반 MCP (환경변수 포함)
claude mcp add --scope user slack \
  -e SLACK_BOT_TOKEN=xoxb-... \
  -e SLACK_TEAM_ID=T0AST... \
  -- npx -y @modelcontextprotocol/server-slack

# Python(uvx) 기반 MCP — 절대경로 필수
claude mcp add --scope user fetch -- /home/gellotin/.local/bin/uvx mcp-server-fetch

# Node 로컬 스크립트
claude mcp add --scope user microsoft-todo \
  -e CLIENT_ID=... \
  -- node /home/gellotin/project/mcp-todo/src/index.js

# 상태 확인
claude mcp list
```

## Atlassian MCP (OAuth 연동)

공식 Rovo MCP는 OAuth 2.0 PKCE 방식으로 인증합니다.

1. `claude mcp add` 로 등록 후 Claude Code 재시작
2. 첫 Atlassian 도구 호출 시 `authenticate` 요청 → 브라우저 URL 발급
3. 브라우저에서 계정 로그인(`gellotin@indg.co.kr`) 및 Confluence/Jira 권한 승인
4. 리다이렉트된 `http://localhost:<port>/callback?code=...` URL을 Claude에 전달 (자동 완료되지 않으면 `complete_authentication` 호출)

**확보된 권한 scope**:
- Confluence: `read/write:page`, `read/write:comment`, `read:space`, `search`, `read:confluence-user`
- Jira: `read:jira-work`, `write:jira-work`

**주의사항**:
- 엔드포인트 `v1/sse`는 2026-06-30 이후 미지원 → `v1/mcp` 사용
- cloudId는 `getAccessibleAtlassianResources`로 조회 후 이후 호출에 재사용 가능
- API 기반이라 정렬/서식 세밀한 제어는 어려움 — 내용은 API, 서식 보정은 브라우저 편집기 권장

## Slack MCP 설정

1. https://api.slack.com/apps → **Create New App** (From scratch)
2. **OAuth & Permissions** → Bot Token Scopes 추가:
   - `chat:write` (메시지 발송)
   - `channels:read` (public 채널 목록 조회)
   - (선택) `groups:read/write`, `im:write`, `channels:manage` 등
3. **Install to Workspace** → 관리자 승인 시 Bot User OAuth Token (`xoxb-...`) 발급
4. 토큰으로 `auth.test` 호출해 `team_id` 확인:
   ```bash
   curl -s -X POST https://slack.com/api/auth.test -H "Authorization: Bearer xoxb-..."
   ```
5. `claude mcp add` 로 `SLACK_BOT_TOKEN` + `SLACK_TEAM_ID` 환경변수와 함께 등록
6. 메시지 발송 대상 채널에 봇 `/invite @앱이름` 필수 (안 하면 `not_in_channel` 에러)

**현재 scope 제약**:
- 기본 `chat:write` + `channels:read`만 설치됨
- 채널 생성(`conversations.create`)은 `channels:manage` 추가 후 **Reinstall to Workspace** 필요
- Private 채널은 `groups:*`, DM은 `im:write` 추가 필요

## 주요 함정 및 트러블슈팅

- **`uvx` PATH 미탐색**: Claude Code는 로그인 셸 PATH를 그대로 쓰지 않음 → Python 기반 MCP는 절대경로(`/home/gellotin/.local/bin/uvx`) 사용
- **npx ≠ PyPI**: `mcp-server-fetch`는 PyPI 전용. `npx @modelcontextprotocol/server-fetch` 형태로 설치 시도하면 404.
- **재시작 필요**: `claude mcp add` 후 새 도구 스키마 로드를 위해 Claude Code 세션 재시작
- **scope 변경 시 재설치**: Slack App은 scope 수정 시 반드시 Reinstall 해야 새 권한이 토큰에 반영됨
- **MCP 상태**: `claude mcp list` 가 가장 빠른 건강 체크. `✓ Connected` / `! Needs authentication` / 에러 메시지 구분

## 보류 중

- **Google Calendar MCP** — Google Cloud Console에서 OAuth Client JSON 발급 필요. 사용 빈도 재검토 후 진행.

## 참고

- Claude Code MCP 공식 문서: https://docs.anthropic.com/claude/docs/claude-code-mcp
- 서버 목록: https://github.com/modelcontextprotocol/servers
- Atlassian Rovo MCP: https://www.atlassian.com/platform/remote-mcp-server
