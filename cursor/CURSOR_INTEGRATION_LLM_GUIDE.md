# Claw-Empire Cursor Agent CLI 통합 가이드 (LLM 적용용)

> **사용법**: 이 문서를 AI 코딩 도우미(Claude, Cursor, ChatGPT 등)에게 주고,
> *"아래 CURSOR_INTEGRATION_LLM_GUIDE.md 내용대로 claw-empire에 Cursor Agent CLI 통합을 적용해 주세요"* 라고 요청하면 됩니다.

대상: [claw-empire](https://github.com/GreenSheep01201/claw-empire) (Climpire)
목표: Cursor Agent CLI를 provider로 추가하여 에이전트가 Cursor CLI로 작업 수행

---

## 요약: 변경할 파일 (26개)

| 파일 | 작업 |
|------|------|
| `.env.example` | CURSOR_API_KEY 주석 블록 추가 |
| `README.md` | Cursor Agent 설치/연결 섹션 추가 |
| `README_ko.md` | Cursor Agent 설치/연결 섹션 추가 |
| `server/modules/routes/core/agents/crud.ts` | cursor를 cli_provider 목록에 추가 |
| `server/modules/routes/ops/models-routes.ts` | fetchCursorModels, CURSOR_MODELS_FALLBACK 추가 |
| `server/modules/workflow/agents/cli-runtime.ts` | CURSOR, CURSOR_AGENT env 제거 |
| `server/modules/workflow/agents/providers/types.ts` | providerKey 필드 추가 |
| `server/modules/workflow/agents/providers/usage-cli-tools.ts` | agent(cursor) 도구 정의 추가 |
| `server/modules/workflow/core/cli-tools.test.ts` | cursor provider 테스트 추가 |
| `server/modules/workflow/core/cli-tools.ts` | cursor case 추가 |
| `server/modules/workflow/orchestration/execution-start-task.ts` | cursor를 provider 목록에 추가 |
| `server/modules/bootstrap/schema/base-schema.ts` | cli_provider CHECK에 cursor 추가 |
| `server/modules/bootstrap/schema/oauth-runtime.ts` | 기존 DB용 cursor CHECK 마이그레이션 추가 |
| `server/modules/workflow/orchestration/report-workflow-tools.ts` | provider 목록에 cursor 추가 |
| `server/modules/routes/core/tasks/execution-run-auto-assign.ts` | VALID_CLI_PROVIDERS에 cursor 추가 |
| `server/modules/routes/collab/office-pack-agent-hydration.ts` | VALID_CLI_PROVIDERS에 cursor 추가 |
| `server/modules/workflow/core/prompt-skills.ts` | cursor를 provider로 추가 |
| `src/components/AgentDetail.tsx` | cursor를 CLI_MODEL_OVERRIDE_PROVIDERS에 추가 |
| `src/components/agent-detail/constants.ts` | cursor 라벨 추가 |
| `src/components/agent-manager/constants.ts` | cursor를 CLI_PROVIDERS에 추가 |
| `src/components/agent-manager/types.ts` | defaultCliProvider prop 추가 |
| `src/components/agent-manager/AgentFormModal.tsx` | CLI_LABELS로 라벨 표시 |
| `src/components/AgentManager.tsx` | defaultCliProvider 사용 |
| `src/components/settings/GeneralSettingsTab.tsx` | cursor option 추가 |
| `src/components/settings/constants.tsx` | cursor CLI_INFO 추가 |
| `src/types/index.ts` | CliProvider 타입 및 DEFAULT_SETTINGS에 cursor 추가 |
| `src/app/office-workflow-pack.ts` | resolveOfficePackSeedProvider에 cursor 추가 |
| `src/app/office-workflow-pack.test.ts` | resolveOfficePackSeedProvider 테스트 수정 (3-provider cycle) |
| `src/app/AppMainLayout.tsx` | AgentManager에 defaultCliProvider 전달 |

---

## 1. `.env.example`

**위치**: `# Optional model provider settings` 직전에 아래 블록 삽입

```env
# Optional: Cursor Agent CLI (Settings > CLI Tools > Cursor Agent). If not set, use `agent login` instead.
# CURSOR_API_KEY="__CHANGE_ME__"

```

---

## 2. `README.md`

**위치**: CLI provider 테이블의 OpenCode 행 다음, `Configure providers and models` 직전

테이블에 행 추가:
```markdown
| [Cursor Agent](https://cursor.com/docs/cli)                   | `curl https://cursor.com/install -fsSL \| bash` | `agent login` or set `CURSOR_API_KEY` |
```

> **마크다운 테이블에서 파이프(`|`) 처리**: 테이블 셀 내부의 `|`는 컬럼 구분자로 해석되어 테이블이 깨질 수 있습니다. `\|`로 이스케이프하거나 HTML 엔티티 `&#124;`를 사용하세요.

그 아래 "How to connect Cursor Agent" 섹션 추가:

```markdown
#### How to connect Cursor Agent

1. **Install the CLI**
   - **macOS / Linux**: Run `curl https://cursor.com/install -fsSL | bash` in a terminal, then add `~/.local/bin` to your PATH as prompted.
   - **Windows**: Run `irm 'https://cursor.com/install?win32=true' | iex` in PowerShell and ensure the `agent` command is on your PATH.

2. **Authenticate (choose one)**
   - **Option A – Login**: Run `agent login` in a terminal and complete the browser/device sign-in.
   - **Option B – API key**: Create an API key from the [Cursor dashboard](https://cursor.com/dashboard?tab=integrations), then set it in one of these places:
     - **Recommended**: In the claw-empire project root **`.env`** file add a line: `CURSOR_API_KEY=your_key` (if you don't have `.env`, copy `.env.example` to `.env` then add the line).
     - Or in the shell: macOS/Linux run `export CURSOR_API_KEY=your_key` or add to `~/.bashrc`/`~/.zshrc`; Windows add `CURSOR_API_KEY` to system environment variables.

3. **Verify**
   - Run `agent status` or `agent --version` in a terminal to confirm install and auth.

4. **Use in Claw-Empire**
   - Open the app and go to **Settings > CLI Tools**; Cursor Agent should show as installed and authenticated.
   - When creating or editing an **agent**, set **CLI provider** to **Cursor Agent** and optionally choose a model, then save.
   - Assign tasks to that agent; they will run via the Cursor Agent CLI.
```

---

## 3. `README_ko.md`

**위치**: CLI 모드 provider 테이블

- 테이블에 Cursor Agent 행 추가 (OpenCode 다음)
- "Cursor Agent 연결 방법" 섹션 추가 (README.md 영문 섹션과 동일한 내용, 한글로)

---

## 4. `server/modules/routes/core/agents/crud.ts`

**변경 1** – `cli_provider` 검증 배열에 `"cursor"` 추가:
```typescript
// 기존
["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "api"]

// 변경 후
["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "cursor", "api"]
```

**변경 2** – `supportsCliModelOverride` 배열에 `"cursor"` 추가:
```typescript
// 기존
["claude", "codex", "gemini", "opencode"]

// 변경 후
["claude", "codex", "gemini", "opencode", "cursor"]
```

---

## 5. `server/modules/routes/ops/models-routes.ts`

**위치**: `function toModelInfo` 다음, `function readModelCache` 직전

추가할 코드:

```typescript
  const CURSOR_MODELS_FALLBACK: CliModelInfoServer[] = [
    { slug: "claude-4.6-opus", displayName: "Claude 4.6 Opus", description: "최고 성능 에이전트, Input $5 / Output $25" },
    { slug: "claude-4.6-sonnet", displayName: "Claude 4.6 Sonnet", description: "SWE-Bench 1위, Input $3 / Output $15" },
    { slug: "gpt-5.3-codex", displayName: "GPT-5.3 Codex", description: "OpenAI 최신 코딩 특화, Input $1.75 / Output $14" },
    { slug: "gemini-3-flash", displayName: "Gemini 3 Flash", description: "가성비 최고, Input $0.5 / Output $3" },
    { slug: "composer-1.5", displayName: "Composer 1.5", description: "Cursor native, Agent/Thinking/Images" },
    { slug: "grok-code", displayName: "Grok Code", description: "초저가 코딩 특화, Input $0.2 / Output $1.5" },
  ];

  async function fetchCursorModels(): Promise<CliModelInfoServer[]> {
    try {
      const output = await execWithTimeout("agent", ["models"], 15_000);
      const lines = output.split(/\r?\n/).map((line: string) => line.trim()).filter(Boolean);
      const seen = new Set<string>();
      const models: CliModelInfoServer[] = [];
      for (const line of lines) {
        const slug = line.replace(/^\s*[-*]\s*/, "").trim();
        if (slug && /^[a-z0-9.-]+$/i.test(slug) && !seen.has(slug)) {
          seen.add(slug);
          models.push({ slug, displayName: slug });
        }
      }
      return models.length > 0 ? models : CURSOR_MODELS_FALLBACK;
    } catch {
      return CURSOR_MODELS_FALLBACK;
    }
  }
```

**주의**: `(l) =>` 대신 `(line: string) =>` 사용 (TypeScript implicit any 방지)

**변경** – `models` 객체에 cursor 추가:
```typescript
opencode: [],
cursor: await fetchCursorModels(),
```

---

## 6. `server/modules/workflow/agents/cli-runtime.ts`

**위치**: `delete cleanEnv.CLAUDE_CODE;` 다음

추가:
```typescript
    delete cleanEnv.CURSOR;
    delete cleanEnv.CURSOR_AGENT;
```

주석은 `// Remove env vars to prevent "nested session" detection when spawning from IDEs` 로 변경해도 됨.

---

## 7. `server/modules/workflow/agents/providers/types.ts`

**위치**: `CliToolDef` 인터페이스 하단

추가:
```typescript
  /** When set, used as the key in CliStatusResult instead of name (e.g. name "agent" -> providerKey "cursor") */
  providerKey?: string;
```

---

## 8. `server/modules/workflow/agents/providers/usage-cli-tools.ts`

**위치**: `CLI_TOOLS` 배열 내 OpenCode 객체 다음

추가:
```typescript
    {
      name: "agent",
      providerKey: "cursor",
      authHint: "Run: agent login or set CURSOR_API_KEY",
      checkAuth: () => {
        if (process.env.CURSOR_API_KEY && process.env.CURSOR_API_KEY.trim()) return true;
        const home = os.homedir();
        const cursorConfig = path.join(home, ".cursor", "cli-config.json");
        if (fileExistsNonEmpty(cursorConfig)) return true;
        const appData = process.env.APPDATA;
        if (appData && fileExistsNonEmpty(path.join(appData, "Cursor", "User", "globalStorage", "cursor-cli-auth.json")))
          return true;
        return false;
      },
    },
```

**변경** – `detectAllCli` 내부 결과 매핑:
```typescript
// 기존
out[CLI_TOOLS[i].name] = results[i];

// 변경 후
const tool = CLI_TOOLS[i];
const key = tool.providerKey ?? tool.name;
out[key] = results[i];
```

---

## 9. `server/modules/workflow/core/cli-tools.test.ts`

**위치**: `describe("buildAgentArgs")` 블록 끝

추가:
```typescript
  it("cursor provider spawns agent with correct flags", () => {
    const tools = createTools();
    const args = tools.buildAgentArgs("cursor", "claude-4-sonnet");

    expect(args[0]).toBe("agent");
    expect(args).toContain("-p");
    expect(args).toContain("--output-format=stream-json");
    expect(args).toContain("--yolo");
    expect(args).toContain("--trust");
    expect(args).toContain("--model");
    expect(args).toContain("claude-4-sonnet");
  });

  it("cursor noTools mode uses --mode=ask", () => {
    const tools = createTools();
    const args = tools.buildAgentArgs("cursor", undefined, undefined, { noTools: true });

    expect(args).toContain("--mode=ask");
  });
```

---

## 10. `server/modules/workflow/core/cli-tools.ts`

**위치**: `case "opencode"` 블록 다음, `case "copilot"` 직전

추가:
```typescript
      case "cursor": {
        const args = ["agent", "-p", "--output-format=stream-json", "--stream-partial-output", "--yolo", "--trust"];
        if (model) args.push("--model", model);
        if (noTools) args.push("--mode=ask");
        return args;
      }
```

---

## 11. `server/modules/workflow/orchestration/execution-start-task.ts`

**위치**: provider 허용 배열

```typescript
// 기존
["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "api"]

// 변경 후
["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "cursor", "api"]
```

---

## 12. `server/modules/bootstrap/schema/base-schema.ts`

**위치**: agents 테이블 CREATE 문의 cli_provider CHECK

```sql
-- 기존
cli_provider TEXT CHECK(cli_provider IN ('claude','codex','gemini','opencode','copilot','antigravity','api')),

-- 변경 후
cli_provider TEXT CHECK(cli_provider IN ('claude','codex','gemini','opencode','copilot','antigravity','cursor','api')),
```

---

## 13. `server/modules/bootstrap/schema/oauth-runtime.ts`

**위치**: api_providers cerebras 마이그레이션 블록 직후, `ALTER TABLE oauth_accounts ADD COLUMN label` 직전

**추가** (기존 DB용 cursor CHECK 마이그레이션):

```typescript
  // cli_provider CHECK 확장: cursor 추가
  try {
    const agentsSql = (db.prepare("SELECT sql FROM sqlite_master WHERE type='table' AND name='agents'").get() as any)
      ?.sql ?? "";
    if (agentsSql && !agentsSql.includes("'cursor'")) {
      const cols = db.prepare("PRAGMA table_info(agents)").all() as Array<{ name: string }>;
      const colNames = cols.map((c) => c.name).filter(Boolean);
      const hasWorkflowPack = colNames.includes("workflow_pack_key");
      db.exec(`
        CREATE TABLE agents_cursor_new (
          id TEXT PRIMARY KEY,
          name TEXT NOT NULL,
          name_ko TEXT NOT NULL DEFAULT '',
          name_ja TEXT NOT NULL DEFAULT '',
          name_zh TEXT NOT NULL DEFAULT '',
          department_id TEXT REFERENCES departments(id),
          workflow_pack_key TEXT NOT NULL DEFAULT 'development',
          role TEXT NOT NULL CHECK(role IN ('team_leader','senior','junior','intern')),
          acts_as_planning_leader INTEGER NOT NULL DEFAULT 0 CHECK(acts_as_planning_leader IN (0,1)),
          cli_provider TEXT CHECK(cli_provider IN ('claude','codex','gemini','opencode','copilot','antigravity','cursor','api')),
          oauth_account_id TEXT,
          api_provider_id TEXT,
          api_model TEXT,
          cli_model TEXT,
          cli_reasoning_level TEXT,
          avatar_emoji TEXT NOT NULL DEFAULT '🤖',
          sprite_number INTEGER,
          personality TEXT,
          status TEXT NOT NULL DEFAULT 'idle' CHECK(status IN ('idle','working','break','offline')),
          current_task_id TEXT,
          stats_tasks_done INTEGER DEFAULT 0,
          stats_xp INTEGER DEFAULT 0,
          created_at INTEGER DEFAULT (unixepoch()*1000)
        )
      `);
      const insertCols =
        "id, name, name_ko, name_ja, name_zh, department_id, workflow_pack_key, role, acts_as_planning_leader, cli_provider, oauth_account_id, api_provider_id, api_model, cli_model, cli_reasoning_level, avatar_emoji, sprite_number, personality, status, current_task_id, stats_tasks_done, stats_xp, created_at";
      const selectCols = hasWorkflowPack
        ? insertCols
        : "id, name, name_ko, name_ja, name_zh, department_id, 'development' AS workflow_pack_key, role, acts_as_planning_leader, cli_provider, oauth_account_id, api_provider_id, api_model, cli_model, cli_reasoning_level, avatar_emoji, sprite_number, personality, status, current_task_id, stats_tasks_done, stats_xp, created_at";
      db.exec(`INSERT INTO agents_cursor_new (${insertCols}) SELECT ${selectCols} FROM agents`);
      db.exec("DROP TABLE agents");
      db.exec("ALTER TABLE agents_cursor_new RENAME TO agents");
      console.log("[Claw-Empire] Migrated agents.cli_provider CHECK to include 'cursor'");
    }
  } catch (e) {
    console.warn("[Claw-Empire] agents cursor migration skipped:", (e as Error)?.message);
  }
```

**중요**: 이 마이그레이션이 없으면 `cli_provider='cursor'` 저장 시 SQLite `constraint failed` (errcode 275) 발생

---

## 14. `server/modules/workflow/orchestration/report-workflow-tools.ts`

**위치**: `COALESCE(cli_provider, '') IN (...)` 목록

```typescript
// 기존 목록에 'cursor' 추가
AND COALESCE(cli_provider, '') IN ('claude','codex','gemini','opencode','copilot','antigravity','cursor','api')
```

---

## 15. `server/modules/routes/core/tasks/execution-run-auto-assign.ts`

**위치**: VALID_CLI_PROVIDERS 선언

```typescript
// 기존
const VALID_CLI_PROVIDERS = new Set(["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "api"]);

// 변경 후
const VALID_CLI_PROVIDERS = new Set(["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "cursor", "api"]);
```

---

## 16. `server/modules/routes/collab/office-pack-agent-hydration.ts`

**위치**: VALID_CLI_PROVIDERS 선언

```typescript
// 기존
const VALID_CLI_PROVIDERS = new Set(["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "api"]);

// 변경 후
const VALID_CLI_PROVIDERS = new Set(["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "cursor", "api"]);
```

---

## 17. `server/modules/workflow/core/prompt-skills.ts`

**변경 1** – PromptSkillProvider 타입:
```typescript
// 기존
type PromptSkillProvider = "claude" | "codex" | "gemini" | "opencode" | "copilot" | "antigravity" | "api";

// 변경 후
type PromptSkillProvider = "claude" | "codex" | "gemini" | "opencode" | "copilot" | "antigravity" | "cursor" | "api";
```

**변경 2** – isPromptSkillProvider:
```typescript
// provider === "cursor" || 조건 추가
```

**변경 3** – getPromptSkillProviderDisplayName:
```typescript
  if (provider === "cursor") return "Cursor Agent";
```

---

## 18. `src/components/AgentDetail.tsx`

```typescript
// 기존
const CLI_MODEL_OVERRIDE_PROVIDERS: Agent["cli_provider"][] = ["claude", "codex", "gemini", "opencode"];

// 변경 후
const CLI_MODEL_OVERRIDE_PROVIDERS: Agent["cli_provider"][] = ["claude", "codex", "gemini", "opencode", "cursor"];
```

---

## 19. `src/components/agent-detail/constants.ts`

`CLI_LABELS` 객체에 추가:
```typescript
  cursor: "Cursor Agent",
```

---

## 20. `src/components/agent-manager/constants.ts`

```typescript
// 기존
export const CLI_PROVIDERS: CliProvider[] = ["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "api"];

// 변경 후
export const CLI_PROVIDERS: CliProvider[] = ["claude", "codex", "gemini", "opencode", "copilot", "antigravity", "cursor", "api"];
```

---

## 21. `src/components/agent-manager/types.ts`

**위치**: AgentManagerProps 인터페이스

```typescript
// defaultCliProvider prop 추가
defaultCliProvider?: import("../../types").CliProvider;
```

---

## 22. `src/components/agent-manager/AgentFormModal.tsx`

**변경 1** – import 추가:
```typescript
import { CLI_LABELS } from "../agent-detail/constants";
```

**변경 2** – CLI provider 버튼 라벨:
```typescript
// 기존: {p}
// 변경 후: {CLI_LABELS[p] ?? p}
```

---

## 23. `src/components/AgentManager.tsx`

**변경 1** – props에 defaultCliProvider 추가

**변경 2** – openCreate 콜백:
```typescript
  const openCreate = useCallback(() => {
    setModalAgent(null);
    setForm({
      ...BLANK,
      cli_provider: defaultCliProvider ?? BLANK.cli_provider,
      department_id: deptTab !== "all" ? deptTab : departments[0]?.id || "",
    });
    setShowModal(true);
  }, [deptTab, departments, defaultCliProvider]);
```

---

## 24. `src/components/settings/GeneralSettingsTab.tsx`

`<option value="opencode">OpenCode</option>` 다음에 추가:
```tsx
            <option value="cursor">Cursor Agent</option>
```

---

## 25. `src/components/settings/constants.tsx`

`CLI_INFO` 객체에 추가 (antigravity 다음):
```tsx
  cursor: { label: "Cursor Agent", icon: "🖥️" },
```

---

## 26. `src/types/index.ts`

**변경 1** – `CliProvider` 타입:
```typescript
// 기존
export type CliProvider = "claude" | "codex" | "gemini" | "opencode" | "copilot" | "antigravity" | "api";

// 변경 후
export type CliProvider = "claude" | "codex" | "gemini" | "opencode" | "copilot" | "antigravity" | "cursor" | "api";
```

**변경 2** – `DEFAULT_SETTINGS.providerModelConfig`에 추가 (opencode 다음, copilot 직전):
```typescript
    cursor: { model: "claude-4-sonnet" },
```

---

## 27. `src/app/office-workflow-pack.ts`

**변경 1** – OfficePackSeedProvider 타입:
```typescript
// 기존
type OfficePackSeedProvider = Extract<CliProvider, "claude" | "codex">;

// 변경 후
type OfficePackSeedProvider = Extract<CliProvider, "claude" | "codex" | "cursor">;
```

**변경 2** – resolveOfficePackSeedProvider 함수:
```typescript
  const cycle: OfficePackSeedProvider[] = ["claude", "codex", "cursor"];
  // planning 부서: cycle[order % cycle.length]
  // 기본 fallback: cycle[seedIndex % cycle.length]
```

**변경 3** – `office-workflow-pack.test.ts` 테스트 업데이트: `resolveOfficePackSeedProvider`를 3-provider cycle로 바꾸면 기존 테스트 "기획팀은 claude/codex를 번갈아 배치한다"가 실패합니다. 아래처럼 수정하세요:
- 테스트 설명: "기획팀은 claude/codex/cursor를 순환 배치한다"로 변경
- `seedOrderInDepartment`: 0 → claude, 1 → codex, 2 → cursor 기대
- (기존 order 1→claude, 2→codex 기대는 새 cycle과 맞지 않아 제거/수정)

---

## 28. `src/app/AppMainLayout.tsx`

**위치**: AgentManager 컴포넌트

```tsx
<AgentManager
  agents={managerAgents}
  departments={managerDepartments}
  defaultCliProvider={settings.defaultProvider}
  onAgentsChange={onAgentsChange}
  ...
/>
```

---

## LLM에 넘길 한 줄 지시 예시

```
아래 CURSOR_INTEGRATION_LLM_GUIDE.md 문서를 읽고, claw-empire 프로젝트에 Cursor Agent CLI를 provider로 통합해 주세요.
문서에 있는 파일 경로와 변경 내용을 그대로 적용해 주시고, **12~13번 DB 스키마/마이그레이션**을 반드시 적용해 주세요.
이 부분이 누락되면 cursor 프로바이더 저장 시 SQLite constraint failed 오류가 발생합니다.
```

---

## 검증

적용 후:

1. `pnpm run build` 성공
2. `pnpm run test:api -- server/modules/workflow/core/cli-tools.test.ts` 성공
3. **`pnpm test`** 전체 통과 (`office-workflow-pack.test.ts` 포함)
4. **서버 재시작** (DB 마이그레이션 적용)
5. `pnpm dev:local` 실행 후 Settings > CLI Tools에서 Cursor Agent 표시
6. 에이전트 생성/편집 시 CLI provider에 "Cursor Agent" 선택 가능
7. 직원에 cursor 할당 후 태스크 실행 시 constraint failed 없이 동작

---

## 주의사항

- **기존 DB 사용 시**: 서버 재시작 후 `[Claw-Empire] Migrated agents.cli_provider CHECK to include 'cursor'` 로그가 한 번 출력되어야 합니다.
- **constraint failed 발생 시**: 12~13번 DB 스키마/마이그레이션 적용 여부를 확인하세요.
- **skill_learning_history**: base-schema의 `skill_learning_history` 테이블 provider CHECK에 cursor가 없을 수 있습니다. Cursor 프로바이더로 스킬 학습이 필요하다면 CHECK에 `'cursor'`를 추가하세요. 스킬 학습을 Cursor에서 지원하지 않는다면 가이드에 별도 명시하세요.
