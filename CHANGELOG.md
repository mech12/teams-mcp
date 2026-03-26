# Changelog

이 포크(`mech12/teams-mcp`)의 변경 이력 및 트러블슈팅 기록.

원본: [floriscornel/teams-mcp](https://github.com/floriscornel/teams-mcp)

---

## 2026-03-26 — 환경변수로 Client ID/Tenant ID 오버라이드 지원 + scope 조정

### 변경 내용

**`src/index.ts`, `src/services/graph.ts`**

- `CLIENT_ID`: `TEAMS_MCP_CLIENT_ID` 환경변수로 오버라이드 가능 (미설정 시 원본 기본값 `14d82eec-...` 유지)
- `AUTHORITY`: `TEAMS_MCP_TENANT_ID` 환경변수로 테넌트 지정 가능 (미설정 시 `/common` 사용)
- `READ_ONLY_SCOPES`에서 `TeamMember.Read.All`, `Chat.Read` 제거
- `FULL_SCOPES`에서 `ChannelMessage.ReadWrite`, `Files.ReadWrite.All` 제거/변경 → `Files.ReadWrite`

### 배경: 왜 커스텀 포크가 필요한가

원본 teams-mcp는 Microsoft 공식 앱 **"Microsoft Graph Command Line Tools"** (`14d82eec-204b-4c2f-b7e8-296a70dab67e`)를 하드코딩 사용합니다.

이 앱의 문제:

1. **동적 권한 요청(incremental consent) 방식** — 앱 등록에 권한이 미리 정의되지 않고, 로그인 시점에 동적으로 요청
2. Azure Portal의 **"Grant admin consent" 버튼이 제대로 동작하지 않음** — 동적 요청 앱에는 이미 등록된 권한(`User.Read` 1개)에 대해서만 동의 부여
3. Microsoft 공식 앱이라 **앱 등록(App registrations)에서 권한을 직접 추가/관리 불가**
4. 관리자가 직접 CLI를 실행하여 인증 + "조직을 대신하여 동의" 체크 필요

### 해결

자사에서 직접 등록한 Azure 앱(`claude-ms-mcp`, `da7a69fc-...`)을 대신 사용:

- 앱 등록에서 **권한을 자유롭게 추가/제거** 가능
- Azure Portal에서 **"Grant admin consent" 버튼이 정상 동작**
- **이미 관리자 동의가 완료**되어 있으므로 추가 관리자 작업 불필요

환경변수로 Client ID와 Tenant를 주입하는 방식으로 구현하여, 원본 기본값도 유지됩니다.

### scope 조정 이유

자사 앱(`claude-ms-mcp`)에 등록/승인된 권한에 맞춰 scope를 조정:

| 원본 scope | 조치 | 이유 |
| --- | --- | --- |
| `TeamMember.Read.All` | 제거 | 자사 앱에 미등록 |
| `Chat.Read` | 제거 | `Chat.ReadWrite`가 포함 |
| `ChannelMessage.ReadWrite` | 제거 | `ChannelMessage.Read.All` + `ChannelMessage.Send`로 충분 |
| `Files.ReadWrite.All` | `Files.ReadWrite`로 변경 | 자사 앱에 `.All` 미등록 |

### 시도했던 방법 (시간순)

1. **adminconsent URL 전달** → 실패 (동적 권한 앱에는 효과 없음)
2. **Azure Portal > Enterprise Applications > Grant admin consent** → 실패 (`User.Read` 1개만 등록되어 있어 나머지 권한 미승인)
3. **Azure Portal > 앱 등록에서 권한 수동 추가** → 불가 (Microsoft 공식 앱이라 수정 불가)
4. **npm 캐시(`~/.npm/_npx/`)에서 Client ID 직접 교체** → 성공했으나 `npx` 재다운로드 시 초기화됨
5. **포크하여 환경변수 오버라이드 지원 추가** → 최종 해결

### 사용법

`~/.claude.json`에 환경변수 설정:

```json
{
  "teams-mcp": {
    "type": "stdio",
    "command": "node",
    "args": ["<클론 경로>/teams-mcp/dist/index.js"],
    "env": {
      "TEAMS_MCP_CLIENT_ID": "da7a69fc-d117-4b16-bf5b-072111ea2c9b",
      "TEAMS_MCP_TENANT_ID": "065a3355-de4f-456c-995a-65341c49e97f"
    }
  }
}
```
