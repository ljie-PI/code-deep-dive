# OpenCode Architecture Summary

> Based on opencode v1.4.6 (commit d7718d41d), ~1421 TypeScript files, ~77,000 lines core package code.
> Language: TypeScript on Bun 1.3.11 runtime. Monorepo with ~20 packages.

---

## 1. Agent Loop

### Core Function: `SessionPrompt.runLoop()` (session/prompt.ts:1315)

The agent loop is a `while(true)` loop (~230 lines) that repeatedly calls the LLM and executes tools until the LLM returns a response with no tool calls.

**Entry chain:**
```
User input
  -> HTTP POST /session/:id/message (server/instance/session.ts)
    -> SessionPrompt.prompt(input) (session/prompt.ts:1286)
      -> createUserMessage(input) -- persist user message
      -> SessionPrompt.runLoop(sessionID)
```

**Loop steps per iteration:**

1. **Load messages**: `MessageV2.filterCompactedEffect(sessionID)` -- loads from SQLite, filters out messages replaced by compaction summaries
2. **Find anchors**: Scan backward for `lastUser`, `lastAssistant`, `lastFinished`, pending `tasks` (compaction/subtask parts)
3. **Exit check**: Break if `lastAssistant` has a finish reason (not "tool-calls"), no pending tool calls, and assistant is newer than user message
4. **Step counter**: `step++`; on step 1, fork background title generation using `title` agent
5. **Task dispatch (priority order)**:
   - Pending subtask -> `handleSubtask()` -> continue
   - Pending compaction -> `compaction.process()` -> continue or break
   - Token overflow detected -> `compaction.create({auto: true})` -> continue
6. **Resolve agent**: Get `Agent.Info`, check max steps limit
7. **Insert reminders**: `insertReminders()` -- inject plan/build-switch prompts if applicable
8. **Create assistant message**: `MessageV2.Assistant` placeholder persisted to DB
9. **Resolve tools**: `resolveTools()` -- registry + MCP + ProviderTransform.schema adaptation
10. **Assemble system prompt**: `[env, skills, instructions]` -- model-specific base prompt + environment info + AGENTS.md files + skill descriptions
11. **Convert messages**: `MessageV2.toModelMessagesEffect()` -- internal format to AI SDK `ModelMessage[]`
12. **Call LLM**: `processor.process()` -> `LLM.stream()` -> Vercel AI SDK `streamText()`
13. **Handle result**: "stop" -> break; "compact" -> create auto compaction -> continue; else continue loop

**Loop exit**: Run `compaction.prune()` in background, return last assistant message.

### Streaming: `SessionProcessor` (session/processor.ts, 617 lines)

The processor receives streaming events from `streamText()` and maps them to persistent Part updates:

| Stream Event | Action |
|---|---|
| `text-start/delta/end` | Create/append/finalize `TextPart` |
| `reasoning-start/delta/end` | Create/append/finalize `ReasoningPart` |
| `tool-input-start/delta/end` | Create `ToolPart` in pending state |
| `tool-call` | Move to running; **doom loop detection** (3 identical consecutive calls -> permission ask) |
| `tool-result` | Complete tool with output |
| `tool-error` | Mark error; set blocked if permission rejected |
| `start-step` | `StepStartPart` with git snapshot |
| `finish-step` | `StepFinishPart` with cost/tokens; check overflow -> set `needsCompaction` |

**Doom loop constant**: `DOOM_LOOP_THRESHOLD = 3` (processor.ts:25)

### LLM Abstraction: `LLM.stream()` (session/llm.ts, 452 lines)

Wraps Vercel AI SDK's `streamText()`. Handles:
- System prompt assembly (base prompt + agent prompt + user system + instructions)
- Tool filtering by permissions
- Provider-specific quirks (OpenAI OAuth, LiteLLM dummy tools, GitLab workflow models)
- `experimental_repairToolCall` -- fixes tool name casing, falls back to `invalid` tool
- AbortController management via `Effect.acquireRelease`

### System Prompt Selection (session/system.ts)

`SystemPrompt.provider(model)` selects base prompt by model ID:

| Pattern | Template |
|---|---|
| `claude*` | `prompt/anthropic.txt` |
| `gpt-4*`, `o1*`, `o3*` | `prompt/beast.txt` |
| `gpt*codex*` | `prompt/codex.txt` |
| `gpt*` (other) | `prompt/gpt.txt` |
| `gemini-*` | `prompt/gemini.txt` |
| `trinity` | `prompt/trinity.txt` |
| `kimi` | `prompt/kimi.txt` |
| default | `prompt/default.txt` |

---

## 2. Tool System

### Tool Definition Interface: `Tool.Def` (tool/tool.ts:36)

```typescript
interface Tool.Def<Parameters extends z.ZodType, M extends Metadata> {
  id: string
  description: string
  parameters: Parameters          // Zod schema
  execute(args, ctx: Tool.Context): Effect<ExecuteResult<M>>
  formatValidationError?(error: z.ZodError): string
}
```

### `Tool.Context` (tool/tool.ts:17)

```typescript
{
  sessionID: SessionID
  messageID: MessageID
  agent: string
  abort: AbortSignal
  callID?: string
  extra?: Record<string, any>     // model info, promptOps for TaskTool
  messages: MessageV2.WithParts[]
  metadata(input): Effect<void>   // update tool call metadata
  ask(input): Effect<void>        // request permission
}
```

### `ExecuteResult` (tool/tool.ts:29)

```typescript
{ title: string, metadata: M, output: string, attachments?: FilePart[] }
```

### Tool Factory: `Tool.define(id, init)`

Lazy initialization. Wraps every tool's execute with automatic Zod parameter validation and output truncation via `Truncate.Service`.

### 19 Built-in Tools

| Tool ID | File | Key Parameters | Description |
|---|---|---|---|
| `bash` | `tool/bash.ts` (510 lines) | `command, timeout?, description?` | PTY shell execution. Default timeout 120s. |
| `read` | `tool/read.ts` | `filePath, offset?, limit?` | Read file, default 2000 lines / 50KB. Images/PDFs as attachments. |
| `write` | `tool/write.ts` | `content, filePath` | Write file. Requires prior read for existing files. Runs formatter + LSP. |
| `edit` | `tool/edit.ts` (688 lines) | `filePath, oldString, newString, replaceAll?` | Exact string replacement with fuzzy correction heuristics. |
| `multiedit` | `tool/multiedit.ts` | `filePath, edits[]` | Sequential multi-edit (delegates to edit). |
| `apply_patch` | `tool/apply_patch.ts` | `patchText` | Multi-file patch. Used instead of edit/write for GPT models. |
| `glob` | `tool/glob.ts` | `pattern, path?` | File pattern match via ripgrep. Max 100 results. |
| `grep` | `tool/grep.ts` | `pattern, path?, include?` | Regex content search via ripgrep. Max 100 matches. |
| `ls` | `tool/ls.ts` | `path?, ignore?` | Tree directory listing. Max 100 files. |
| `webfetch` | `tool/webfetch.ts` | `url, format?, timeout?` | HTTP fetch with HTML-to-markdown. Max 5MB, 120s. |
| `websearch` | `tool/websearch.ts` | `query, numResults?` | Exa web search. opencode provider or OPENCODE_ENABLE_EXA only. |
| `codesearch` | `tool/codesearch.ts` | `query, tokensNum?` | Exa code search. Same availability as websearch. |
| `task` | `tool/task.ts` | `description, prompt, subagent_type?, task_id?, command?` | Spawn subagent session. |
| `question` | `tool/question.ts` | `questions[]` | Ask user questions. app/cli/desktop clients only. |
| `skill` | `tool/skill.ts` | `name` | Load specialized skill instructions. |
| `todowrite` | `tool/todo.ts` | `todos[]` | Update session todo list. |
| `lsp` | `tool/lsp.ts` | `operation, filePath, line, character` | LSP operations (experimental). |
| `plan_exit` | `tool/plan.ts` | (none) | Exit plan mode (experimental, CLI only). |
| `invalid` | `tool/invalid.ts` | `tool, error` | Placeholder for invalid tool calls. |

### ToolRegistry (tool/registry.ts, 345 lines)

- `tools(model, agent)`: Returns filtered tools based on agent permissions, provider capabilities, and model type
- GPT models (not GPT-4, not OSS) get `apply_patch` instead of `edit`+`write`
- Task tool description dynamically enriched with available subagent types
- Skill tool description dynamically enriched with available skills

### Output Truncation (tool/truncate.ts)

Constants: `MAX_LINES = 2000`, `MAX_BYTES = 50KB`, retention 7 days. Over-limit output written to temp file; truncated preview returned with hint to use Read/Grep/Task for full content.

---

## 3. Subagents / Multi-Agent

### Task Tool (tool/task.ts)

The primary mechanism for multi-agent behavior. When the main agent calls the `task` tool:

1. Creates a `SubtaskPart` on the current user message with `{prompt, description, agent, model?, command?}`
2. The main `runLoop()` detects the subtask in the next iteration
3. `handleSubtask()` spawns a child session:
   - Creates a new session (child of current)
   - Routes to the specified agent (e.g., "general", "explore", or custom)
   - Inherits model from parent unless agent overrides
   - Runs the subagent's own `runLoop()` in its own context
   - Returns results back to the parent session

### Built-in Agent Types for Subagents

| Agent | Mode | Tools | Purpose |
|---|---|---|---|
| `general` | subagent | Full tool access (except todowrite) | Multi-step task execution |
| `explore` | subagent | Read-only (grep, glob, list, bash, read, webfetch) | Fast codebase exploration |

### Agent Configuration: `Agent.Info` (agent/agent.ts:28)

```typescript
{
  name: string
  mode: "subagent" | "primary" | "all"
  permission: Permission.Ruleset
  model?: { providerID, modelID }
  variant?: string
  prompt?: string
  options: Record<string, any>
  steps?: number                    // max agent loop iterations
  temperature?: number
  topP?: number
  hidden?: boolean
  native?: boolean
}
```

### 7 Built-in Agents

| Agent | Mode | Purpose |
|---|---|---|
| `build` | primary | Default coding agent. Full tool access. |
| `plan` | primary | Plan mode. Restricted editing (only .opencode/plans/*.md). |
| `general` | subagent | Multi-step task execution. |
| `explore` | subagent | Read-only codebase exploration. |
| `compaction` | primary (hidden) | Conversation summarization. No tools. |
| `title` | primary (hidden) | Title generation. Temperature 0.5. No tools. |
| `summary` | primary (hidden) | Summary generation. No tools. |

### Agent Auto-Generation

`Agent.Service.generate()` uses LLM (`generateObject()`) to create agent configs from natural language descriptions. Returns `{identifier, whenToUse, systemPrompt}`.

---

## 4. Skills

### What Are Skills?

Skills are `SKILL.md` files that provide domain-specific instructions to the agent. They are discoverable command-like resources that contain specialized knowledge.

### Skill Discovery (skill/ directory, 2 files)

- Skills are discovered from configurable paths and URLs (`config.skills`)
- Also from Claude Code skill directories (unless `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS`)
- External skills from `OPENCODE_DISABLE_EXTERNAL_SKILLS`

### Skill Execution

1. Agent calls the `skill` tool with `{name: "skill-name"}`
2. Skill tool loads the SKILL.md file content
3. Returns content in `<skill_content>` XML block with bundled files list
4. The skill instructions are injected into the conversation as tool output
5. Agent follows the skill's instructions for subsequent actions

### Skill in System Prompt

`SystemPrompt.skills(agent)` generates a description of available skills that is included in the system prompt, so the LLM knows which skills it can invoke.

---

## 5. Context Management

### Architecture

Context management uses a three-stage approach: **overflow detection -> pruning -> compaction (summarization)**.

### Key Constants

| Constant | Value | Location |
|---|---|---|
| `COMPACTION_BUFFER` | 20,000 tokens | overflow.ts:4 |
| `PRUNE_MINIMUM` | 20,000 tokens | compaction.ts:33 |
| `PRUNE_PROTECT` | 40,000 tokens | compaction.ts:34 |
| `PRUNE_PROTECTED_TOOLS` | `["skill"]` | compaction.ts |
| `DOOM_LOOP_THRESHOLD` | 3 | processor.ts:25 |

### Overflow Detection (`SessionCompaction.isOverflow`, overflow.ts)

```
count = total || (input + output + cache.read + cache.write)
reserved = config.compaction.reserved ?? min(COMPACTION_BUFFER, maxOutputTokens)
usable = model.limit.input ? (limit.input - reserved) : (context - maxOutputTokens)
overflow = count >= usable
```

Disabled if `config.compaction.auto === false` or model context is 0.

### Pruning (compaction.ts:91)

Before full compaction, tries to free space by clearing old tool outputs:
1. Walk messages backward from newest
2. Skip first 2 user turns (protect recent context)
3. Stop at compaction summaries
4. Accumulate token estimates for completed tool outputs
5. Once past `PRUNE_PROTECT` (40K tokens), mark excess tools for pruning
6. If pruned total > `PRUNE_MINIMUM` (20K tokens): set `compacted` timestamp on tool outputs
7. Compacted outputs replaced with `[Old tool result content cleared]` during model message conversion

### Full Compaction (compaction.ts:139)

When pruning is insufficient:
1. Find compaction boundary (user message)
2. Use `compaction` agent with dedicated prompt template
3. Stream LLM response (no tools) to generate summary
4. Create assistant message with `summary: true`
5. All messages before the summary are filtered out by `filterCompactedEffect()`
6. If overflow during streaming: create auto-compaction and replay

### Compaction Prompt Template

```
Provide a detailed prompt for continuing our conversation above.
Focus on information that would be helpful for continuing...
[Sections: Goal, Instructions, Discoveries, Accomplished, Relevant files]
```

### Auto vs Manual

| Type | Trigger | Flag |
|---|---|---|
| Auto | Token overflow detection in runLoop | `auto: true` |
| Manual | User API call `/session/:id/compact` | `auto: false` |
| Overflow | LLM response truncated mid-stream | `overflow: true` |

Disableable via `OPENCODE_DISABLE_AUTOCOMPACT` env var or `config.compaction.auto: false`.

---

## 6. Permission System

### Core Algorithm: `evaluate()` (permission/evaluate.ts, 15 lines)

```typescript
function evaluate(permission, pattern, ...rulesets): Rule {
  const rules = rulesets.flat()
  const match = rules.findLast(
    (rule) => Wildcard.match(permission, rule.permission) &&
              Wildcard.match(pattern, rule.pattern)
  )
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

**Key behavior**: Last-match-wins with dual wildcard matching. Default action is "ask".

### Rule Structure

```typescript
{ permission: string, pattern: string, action: "allow" | "deny" | "ask" }
```

### Rule Sources (merged in order, lowest to highest priority)

1. Agent default permissions (`agent.permission`)
2. Session-level permission overrides (`session.permission`)
3. Runtime-approved rules (in-memory `approved` set, persisted to `PermissionTable`)

### Permission Flow

1. Tool calls `ctx.ask({permission, patterns, ruleset, always, metadata})`
2. Each pattern evaluated against merged rulesets
3. **deny** -> `DeniedError` (immediate)
4. **allow** -> pass through
5. **ask** -> Creates `Deferred`, publishes `permission.asked` event, UI shows dialog
6. User replies: "once" (allow this time) | "always" (add to approved, auto-resolve matching pending) | "reject" (`RejectedError` or `CorrectedError` with feedback)

### Batch Resolution

When user chooses "always", all pending requests in the same session that now match approved rules are auto-resolved. When user rejects, all pending requests in the same session are rejected.

### Error Types

| Error | Cause |
|---|---|
| `DeniedError` | Rule explicitly denies |
| `RejectedError` | User rejected (no feedback) |
| `CorrectedError` | User rejected with feedback text (fed back to LLM) |

### Default Permission Ruleset (agent/agent.ts:88-105)

```
*: allow
doom_loop: ask
external_directory: { *: ask, <truncation_dir>: allow, <skill_dirs>: allow }
question: deny
plan_enter: deny
plan_exit: deny
read: { *: allow, *.env: ask, *.env.*: ask, *.env.example: allow }
```

### Bash Command Arity (permission/arity.ts)

`BashArity.prefix(tokens)` extracts command identity for permission matching. E.g., `["git", "checkout", "main"]` -> pattern `"git checkout"`. Covers 100+ commands with known arities.

---

## 7. MCP Integration

### Architecture

OpenCode acts as an MCP **client**. `MCP.Service` (mcp/index.ts, 930 lines) manages connections to external MCP servers.

### Transport Types

| Type | Config | Implementation |
|---|---|---|
| local (stdio) | `type: "local"` | `StdioClientTransport` -- spawns subprocess |
| remote (HTTP) | `type: "remote"` | `StreamableHTTPClientTransport` (tries first) |
| remote (SSE) | `type: "remote"` | `SSEClientTransport` (fallback) |

### MCP Config

```json
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ["node", "server.js"],
      "env": { "KEY": "value" },
      "timeout": 30000
    },
    "remote-server": {
      "type": "remote",
      "url": "https://mcp.example.com",
      "headers": {},
      "oauth": {}
    }
  }
}
```

Default timeout: `30,000ms`.

### Tool Conversion (`convertMcpTool`)

MCP tools are converted to AI SDK format:
- Input schema normalized to `type: "object"` with `additionalProperties: false`
- Tool name namespaced: `{sanitized_server}_{sanitized_tool}`
- Execute calls `client.callTool()` with timeout
- All MCP tool calls require permission confirmation by default

### OAuth Support

Full OAuth 2.0 flow: `McpOAuthProvider` (token management), `McpOAuthCallback` (local HTTP callback server). Supports dynamic client registration, pre-registered client IDs, token refresh, PKCE.

### Additional MCP Features

- `prompts()` / `getPrompt()` -- list/fetch prompts from servers
- `resources()` / `readResource()` -- list/read resources
- `ToolListChangedNotification` -- dynamic tool updates

---

## 8. Configuration

### Multi-Layer Merge (lowest to highest priority)

1. Well-known remote configs (`/.well-known/opencode`)
2. Global config (`~/.config/opencode/{config.json, opencode.json, opencode.jsonc}`)
3. Custom config via `OPENCODE_CONFIG` env var
4. Project configs (`opencode.json`/`opencode.jsonc` walking up to worktree root)
5. Directory configs (`.opencode/opencode.json`)
6. `OPENCODE_CONFIG_CONTENT` env var (inline JSON)
7. Console org config
8. Managed directory (`/etc/opencode/`, `/Library/Application Support/opencode/`, `%ProgramData%\opencode`)
9. macOS Managed Preferences (MDM `.mobileconfig`, highest priority)
10. CLI flags

Arrays like `instructions` and `plugins` are concatenated, not replaced.

### Config Schema Core Fields

| Field | Type | Description |
|---|---|---|
| `provider` | `Record<string, ProviderConfig>` | LLM provider configs (API keys, base URLs) |
| `model` | `string` | Default model (`provider/model` format) |
| `agent` | `Record<string, AgentConfig>` | Agent overrides |
| `mcp` | `Record<string, McpConfig>` | MCP server configs |
| `plugin` | `PluginSpec[]` | Plugin specifiers |
| `permission` | `Permission` | Default permission rules |
| `snapshot` | `boolean` | Git snapshot enable/disable |
| `compaction` | `{auto?, prune?, reserved?}` | Compaction settings |
| `instructions` | `string[]` | Additional instruction file paths |
| `formatter` | `false | Record<string, FormatterConfig>` | Formatter configs |
| `lsp` | `false | Record<string, LspConfig>` | LSP server configs |
| `skills` | `Skills` | Additional skill paths/URLs |

### AGENTS.md / Instruction System (session/instruction.ts)

- Walks up from working directory looking for `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`
- Reads global files: `$OPENCODE_CONFIG_DIR/AGENTS.md`, `~/.opencode/AGENTS.md`, `~/.claude/CLAUDE.md`
- Supports `@./path/to/file` references (expanded to file content)
- Supports `` !`shell_command` `` syntax for shell interpolation
- Per-directory AGENTS.md discovered dynamically when files are read during tool calls

### Config File Format

JSONC (JSON with comments) via `jsonc-parser`. Updates preserve comments and formatting.

### Key Files

| File | Lines | Role |
|---|---|---|
| `config/config.ts` | 1661 | Config core: Schema, merge, read/write |
| `config/paths.ts` | 167 | Platform path resolution, JSONC parsing |
| `config/markdown.ts` | 99 | Markdown frontmatter parsing for agents/commands |
| `session/instruction.ts` | 243 | AGENTS.md / instruction file loading |

---

## 9. Session/Message Model

### Three-Layer Model: Session -> Message -> Part

**Session** (session/index.ts, 818 lines):
- `id: SessionID` (ULID, descending for newest-first)
- `slug: string` (URL-friendly, e.g., "happy-tiger")
- `projectID: ProjectID`
- `directory: string` (working directory)
- `parentID?: SessionID` (for forked sessions)
- `title: string`
- `permission?: Permission.Ruleset` (session-level overrides)
- `summary?: {additions, deletions, files, diffs?}`
- `revert?: {messageID, partID?, snapshot?, diff?}`
- `time: {created, updated, compacting?, archived?}`

**Message** (session/message-v2.ts, 1057 lines):
- Two roles: `User` and `Assistant`
- User: `{agent, model: {providerID, modelID, variant?}, format?, tools?}`
- Assistant: `{agent, mode, path: {cwd, root}, cost, tokens: {input, output, reasoning, cache: {read, write}}, modelID, providerID, finish?, summary?, structured?, error?}`

**Part types (12)**:

| Type | Key Fields | Purpose |
|---|---|---|
| `text` | text, synthetic?, ignored? | Text content |
| `reasoning` | text, signature? | LLM reasoning traces |
| `tool` | callID, tool, state (pending/running/completed/error) | Tool call lifecycle |
| `file` | url, mime, filename? | File attachments |
| `subtask` | prompt, description, agent, model?, command? | Subagent task |
| `compaction` | auto, overflow? | Compaction boundary marker |
| `step-start` | snapshot? | Agent step begin |
| `step-finish` | reason, snapshot?, cost, tokens | Agent step end |
| `snapshot` | hash | Git snapshot ref |
| `patch` | hash, files[] | File change patch |
| `agent` | name | Agent reference |
| `retry` | attempt, error, time | Retry record |

### ID System

All IDs are ULIDs (Universally Unique Lexicographically Sortable Identifiers) with branded types:
- `SessionID`, `MessageID`, `PartID`, `ProjectID`, `WorkspaceID`, `PermissionID`
- `ascending()` for chronological ordering, `descending()` for newest-first

### Persistence

SQLite via Drizzle ORM. Tables: `session`, `message`, `part`, `todo`, `session_entry`, `permission`, `project`.

Message and Part data stored as JSON in `data` column. Part table includes denormalized `session_id` for query efficiency.

### Message Conversion: `toModelMessagesEffect()` (message-v2.ts:585)

Converts internal format to Vercel AI SDK `ModelMessage[]`:
- Text parts -> `{type: "text", text}`
- File parts (images) -> `{type: "image", image: URL}`
- Tool parts (completed) -> tool_call + tool_result pair
- Tool parts (compacted) -> `[Old tool result content cleared]`
- Tool parts (error/interrupted) -> error result
- Media injection: For providers that don't support media in tool results, media collected and injected as synthetic user message

---

## 10. Server/API

### Architecture

Hono HTTP server (server/server.ts, ~109 lines) with embedded WebSocket support. Runs in two modes:

- **Embedded mode**: Started by TUI command, CLI/TUI communicates via SDK
- **Headless mode**: `opencode serve --port 3000`

### Worker Thread Architecture (TUI mode)

- **Main thread**: SolidJS + opentui terminal UI
- **Worker thread**: Hono server + Effect runtime + agent engine
- Communication via RPC (`Rpc.client`/`Rpc.emit`)

### Middleware Chain

1. `ErrorMiddleware` -- maps NamedError to HTTP status
2. `AuthMiddleware` -- Bearer token via `OPENCODE_SERVER_PASSWORD`
3. `LoggerMiddleware` -- request timing
4. `CompressionMiddleware` -- gzip (skips SSE endpoints)
5. `CorsMiddleware` -- localhost, tauri origins, *.opencode.ai

### Key API Routes

| Category | Route | Method | Purpose |
|---|---|---|---|
| Session | `/session` | GET/POST | List/create sessions |
| Session | `/session/:id/message` | GET/POST | List messages / send prompt |
| Session | `/session/:id/abort` | POST | Cancel running prompt |
| Session | `/session/:id/fork` | POST | Fork session |
| Session | `/session/:id/revert` | POST | Revert to snapshot |
| Permission | `/permission` | GET | List pending permissions |
| Permission | `/permission/:id/reply` | POST | Reply to permission request |
| Provider | `/provider` | GET | List providers |
| MCP | `/mcp` | GET/POST | Status / add server |
| Event | `/event` | GET (WS) | WebSocket event stream |
| PTY | `/pty` | CRUD + WS | Terminal management |
| TUI | `/tui/*` | POST | TUI control (prompt, commands, toasts) |

### Event Streaming

Two-tier:
- **Instance SSE** (`/event`): Subscribes to instance `Bus`, pushes all events. 10s heartbeat.
- **Global SSE** (`/global/event`): Subscribes to `GlobalBus`, wraps events with directory/project/workspace metadata.

### OpenAPI + SDK Generation

Server uses `hono-openapi` to generate OpenAPI spec. SDK auto-generated by `@hey-api/openapi-ts` into `packages/sdk/js/`. v2 SDK: types (5338 lines), client (4274 lines).

### Runtime Adapter

Conditional imports (`#hono`) for Bun vs Node:
- Bun: `bun:sqlite`, `Bun.serve()`, `createBunWebSocket()`
- Node: `@hono/node-server`, `@hono/node-ws`, `better-sqlite3`

---

## 11. UI

### TUI (Terminal UI)

**Framework**: SolidJS + `@opentui/solid` (renders SolidJS to terminal)

**Provider tree** (22 nested providers):
```
ErrorBoundary -> ArgsProvider -> ExitProvider -> KVProvider -> ToastProvider
  -> RouteProvider -> TuiConfigProvider -> SDKProvider -> ProjectProvider
    -> SyncProvider -> ThemeProvider -> LocalProvider -> KeybindProvider
      -> PromptStashProvider -> DialogProvider -> CommandProvider
        -> FrecencyProvider -> PromptHistoryProvider -> PromptRefProvider -> App
```

**Key components**:

| Component | File | Lines | Role |
|---|---|---|---|
| `App` | `tui/app.tsx` | 864 | Root, routing, global state |
| `SessionRoute` | `tui/routes/session/index.tsx` | 2288 | Message list, tool output, permissions |
| `Prompt` | `tui/component/prompt/index.tsx` | 1274 | Input with autocomplete, history, file attach |
| `Theme` | `tui/context/theme.tsx` | 1238 | 30+ built-in themes |
| `Permission` | `tui/routes/session/permission.tsx` | 691 | Permission dialog |

**Routing**: Three routes -- Home (logo + prompt), Session (conversation), Plugin (custom).

**State sync**: `context/sync.tsx` (516 lines) maintains SolidJS store synced from SSE events.

### Web App (packages/app)

**Framework**: SolidStart + TailwindCSS 4.x + TanStack Query

**Key pages**: `layout.tsx` (2505 lines), `session.tsx` (2062 lines), `prompt-input.tsx` (1575 lines)

### Shared UI Library (packages/ui)

`@opencode-ai/ui` shared between TUI and Web:
- `message-part.tsx` (2326 lines) -- renders all Part types (markdown, syntax highlighting via Shiki, diffs, images, tool outputs)

### Desktop

- **Tauri** (`packages/desktop`): Rust + WebView, reuses web app
- **Electron** (`packages/desktop-electron`): In-process server, reuses web app

---

## 12. Plugin System

### Plugin Interface (`@opencode-ai/plugin`)

```typescript
interface PluginDefinition {
  name: string
  tools?: ToolDefinition[]
  agents?: AgentDefinition[]
  commands?: CommandDefinition[]
  providers?: ProviderDefinition[]
  hooks?: {
    "tool.execute.before"?: (context, data) => void
    "tool.execute.after"?: (context, data) => void
    "command.execute.before"?: (context, data) => void
    "experimental.chat.messages.transform"?: (context, data) => void
    "experimental.chat.system.transform"?: (context, data) => void
    "experimental.text.complete"?: (context, data) => void
    "experimental.session.compacting"?: (context, data) => void
    "experimental.compaction.autocontinue"?: (context, data) => void
  }
}
```

### Plugin Sources

1. **npm packages**: Resolved via `npa` + installed via `Npm.add()`
2. **[package, options] tuples**: npm with config
3. **File paths**: Local `.ts`/`.js` files
4. **Built-in**: Codex, Copilot, GitLab, Poe, Cloudflare

### Plugin Loading Pipeline

1. **Plan**: Parse `PluginSpec` from config
2. **Resolve**: Find target, check `engines.opencode` compatibility, resolve entrypoint (`exports["./server"]` or `main`)
3. **Load**: Dynamic `import()` of entrypoint
4. **Apply**: Call plugin function with `PluginInput` context (SDK client, project info, worktree, directory)

### TUI Plugin System

Separate TUI plugin runtime (1031 lines) with:
- 9 internal feature plugins (sidebar panels, footer, tips, etc.)
- Slot system for named insertion points (`home_logo`, `home_prompt`, `home_footer`, etc.)
- Full API: commands, routes, themes, events, KV storage, renderer access

### Plugin Initialization

`Plugin.init()` runs first in `InstanceBootstrap`. Loads and activates all plugins before other subsystems.

---

## 13. Tech Stack

| Area | Technology |
|---|---|
| **Runtime** | Bun 1.3.11 (Node.js compatible) |
| **Language** | TypeScript 5.8.2 |
| **Package Management** | Bun workspaces + Turborepo |
| **Dependency Injection** | Effect 4.0.0-beta (`Context.Service` + `Layer` pattern) |
| **LLM SDK** | Vercel AI SDK 6.x (`streamText`, `generateText`, `tool()`) |
| **HTTP Server** | Hono 4.x |
| **Database** | SQLite (Drizzle ORM), WAL mode, 64MB cache |
| **Terminal UI** | SolidJS + `@opentui/solid` |
| **Web Frontend** | SolidJS + SolidStart + TailwindCSS 4.x |
| **Desktop** | Tauri 2.x (Rust) + Electron |
| **Schema Validation** | Zod 4.x |
| **Tool Protocol** | MCP (Model Context Protocol) |
| **Agent Protocol** | ACP (Agent Client Protocol) |
| **Code Intelligence** | LSP (Language Server Protocol) |
| **Process Execution** | PTY (pseudo-terminal) via cross-spawn |
| **File Search** | ripgrep |
| **VCS** | Git (separate snapshot repo) |
| **Observability** | OpenTelemetry |
| **Config Format** | JSONC (json-parser) |
| **ID Generation** | ULID |

### Key Libraries

- `@ai-sdk/*` -- 23 bundled provider SDKs
- `drizzle-orm` -- SQLite ORM
- `yargs` -- CLI framework
- `gray-matter` -- YAML frontmatter parsing
- `@modelcontextprotocol/sdk` -- MCP client
- `@agentclientprotocol/sdk` -- ACP support
- `bonjour-service` -- mDNS discovery

### Codebase Scale

- ~1421 TypeScript files (full monorepo)
- ~350 .ts/.tsx files in core package
- ~77,441 lines in core package
- ~40 subdirectories in `packages/opencode/src/`
- Largest file: 2,288 lines (TUI SessionRoute)
- Median file: ~100 lines

---

## 14. Design Philosophy

### Principle 1: Effect as Unified Computation Model

All code uses Effect library instead of async/await:
- `yield* SomeService` for dependency injection
- `Scope` + `addFinalizer` for resource management
- `Effect.all`, `Effect.forEach` for structured concurrency
- `Effect<A, E, R>` for typed errors
- Layer replacement for testing

### Principle 2: Namespace Pattern

Every module uses TypeScript `namespace` to organize types, services, and implementations:
```typescript
export namespace Session {
  export const Info = z.object({...})
  export type Info = z.infer<typeof Info>
  export class Service extends Context.Service<...>()("@opencode/Session") {}
  export const layer = Layer.effect(Service, ...)
}
```

### Principle 3: Service + Layer + InstanceState

Three-piece pattern for every subsystem:
1. `Interface` -- method signatures
2. `Service` -- Effect Context Tag (`@opencode/ModuleName`)
3. `Layer` -- implementation with declared dependencies
4. `InstanceState` -- per-project-directory mutable state (lazily initialized, auto-cleaned)

### Principle 4: Event-Driven Decoupling

Two-tier event bus:
- **Instance Bus** (`Bus.Service`): Effect PubSub, per-instance scope
- **Global Bus** (`GlobalBus`): Node.js EventEmitter, cross-instance

Events defined with `BusEvent.define(type, zodSchema)` for type safety. Used for: permission flow, session status, message streaming deltas, config changes, UI updates.

### Principle 5: Projector Pattern (Partial Event Sourcing)

Messages/parts persisted via SyncEvents projected to SQLite:
- `session.created` -> INSERT SessionTable
- `message.updated` -> UPSERT MessageTable
- `message.part.updated` -> UPSERT PartTable

### Principle 6: Data-Driven Configuration

Everything externalized: providers, agents, permissions, tools, MCP servers, plugins. 10-layer merge with managed config at highest priority for enterprise MDM.

### Principle 7: Provider Abstraction with Transform Layer

`ProviderTransform` (1067 lines) normalizes across 23+ providers:
- Schema sanitization (Gemini JSON Schema restrictions)
- Message normalization (Anthropic empty content, Mistral tool call IDs)
- Cache control hints (per-provider keys)
- Temperature/topP/topK defaults per model family
- Reasoning variant generation per provider SDK
- Output token management (`OUTPUT_TOKEN_MAX = 32,000`)

### Key Non-Traditional Choices

1. **Zod schemas as source of truth** (not TypeScript interfaces) -- runtime validation + JSON Schema generation
2. **SQLite over filesystem** -- ACID transactions, concurrent access, query flexibility
3. **Separate Git repo for snapshots** -- non-invasive, independent of user's Git workflow
4. **SolidJS for terminal UI** -- same component model for TUI and Web
5. **PTY for shell execution** -- real terminal support (ANSI, interactive programs)
6. **Worker thread for TUI** -- server and UI never block each other
7. **Conditional imports** (`#db`, `#pty`, `#hono`) -- single codebase for Bun and Node

### Architectural Constraints

| Constraint | Reason | Impact |
|---|---|---|
| Bun runtime | Performance + native SQLite | Node.js needs adapter layer |
| Effect library | DI + resource management | Steep learning curve |
| Single-process server | Simplified architecture | One Instance per directory |
| SQLite | Embedded database | Not suited for multi-machine |
| Git CLI dependency | Snapshot implementation | Requires system Git |
