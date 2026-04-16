# Claude Code Architecture Summary

Based on reverse-engineering documentation of Claude Code v2.1.888 (~546K lines of TypeScript, ~3,612 files).

---

## 1. Agent Loop

### Core Structure
- **Entry**: `src/entrypoints/cli.tsx` (322 lines) -> `src/main.tsx` (6,603 lines) -> `src/entrypoints/init.ts` (344 lines)
- **Three execution modes**: Interactive REPL, Headless (`-p`/`--print`), SDK (`--sdk-url`)

### Three-Layer Architecture
| Layer | File | Lifetime | Core State |
|-------|------|----------|------------|
| **Session** | `src/QueryEngine.ts` (1,320 lines) | Entire session | `mutableMessages`, `totalUsage`, `readFileState`, `permissionDenials` |
| **Turn** | `src/query.ts` (1,732 lines) | Single user request | `State` object (messages, toolUseContext, turnCount, transition, etc.) |
| **Tool Execution** | `StreamingToolExecutor.ts` (530 lines) | Single API response | `TrackedTool` queue, concurrency state, `siblingAbortController` |

### The Main Loop
- `query()` is an **`async function*`** (AsyncGenerator)
- Internal `queryLoop()` runs a **`while(true)`** loop
- Each iteration passes through checkpoints:
  1. **Tool Result Budget** - truncate oversized tool results
  2. **Snip** - trim early history messages
  3. **Microcompact** - fold duplicate file reads
  4. **Context Collapse** - replace operation sequences with summaries (feature-gated)
  5. **Autocompact** - full summary compression when tokens exceed ~90% of model limit
  6. **Setup** - build request params
  7. **API call** - streaming via `deps.callModel()`
  8. **Tool execution** - via `StreamingToolExecutor`
  9. **Continue/Terminal decision**

### State Object (per-turn)
```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

### Terminal Conditions (10 reasons)
`completed`, `aborted_streaming`, `aborted_tools`, `max_turns`, `prompt_too_long`, `image_error`, `model_error`, `blocking_limit`, `hook_stopped`, `stop_hook_prevented`

### Continue Transitions (7 reasons)
`next_turn`, `collapse_drain_retry`, `reactive_compact_retry`, `max_output_tokens_escalate`, `max_output_tokens_recovery`, `stop_hook_blocking`, `token_budget_continuation`

### Error Recovery Strategy
- Gradient approach: lightest recovery first, escalate to heavier
- 413 errors: Context Collapse drain -> Reactive Compact -> yield error
- max_output_tokens: escalate 8K -> 64K -> multi-turn continuation (max 3 times)
- Autocompact circuit breaker: `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`

---

## 2. Tool System

### Tool Interface
- `src/Tool.ts` (792 lines) - **47 fields/methods** across 6 domains:
  - **Identity**: `name`, `aliases`, `searchHint`, `mcpInfo`
  - **Schema**: `inputSchema` (Zod), `inputJSONSchema`, `outputSchema`, `strict`
  - **Core Execution**: `call`, `validateInput`, `checkPermissions`, `description`, `prompt`
  - **Behavior Flags**: `isConcurrencySafe`, `isReadOnly`, `isDestructive`, `isEnabled`, `interruptBehavior`
  - **UI Rendering**: `renderToolUseMessage`, `renderToolResultMessage`, etc.
  - **Auxiliary**: `inputsEquivalent`, `getPath`, `maxResultSizeChars`, etc.

### Tool Registration Pipeline (3 layers in `src/tools.ts`, 387 lines)
1. **`getAllBaseTools()`** - single source of truth, up to 54 tools (conditional loading via feature flags, env vars, runtime checks)
2. **`getTools(permissionContext)`** - mode filtering, deny rules, enabled checks
3. **`assembleToolPool()`** - merge built-in + MCP tools, partition-sort (built-in prefix for cache stability), `uniqBy` dedup (built-in wins)

### Tool Factory
- `buildTool(def)` fills defaults from `TOOL_DEFAULTS` (**fail-closed** principle):
  - `isConcurrencySafe: false` (assumes unsafe)
  - `isReadOnly: false` (assumes side effects)
  - `checkPermissions: { behavior: 'allow' }` (delegates to general system)

### Tool Count
- 52 tool directories in `src/tools/`
- 54 registered tools (some directories register multiple)
- Categories: File ops (4), Shell (2), Search (4), Agent/Task (10), Cron (3), Plan/Workflow (5), Web (3), MCP (4), Interaction (14+), Worktree (2), Internal/Test (8+)

### Key Tools
- **BashTool**: 2,592 lines of security checks (`bashSecurity.ts`), 2,621 lines of permissions (`bashPermissions.ts`), tree-sitter AST parsing, sandbox support, auto-backgrounding after 15s
- **FileEditTool**: search-and-replace strategy, 14-step validation pipeline, uniqueness constraint, quote normalization, mtime-based external modification detection
- **FileReadTool**: 6 output types, 25K token budget, mtime-based dedup (~18% hit rate), 3-level image compression, PDF dual-path handling
- **AgentTool**: 1,834 lines, fork/normal paths, sync/async modes, 4-layer tool filtering

### Tool Execution Lifecycle (`toolExecution.ts`, 1,745 lines)
1. `findToolByName()` - lookup by name or alias
2. `inputSchema.safeParse()` - Zod validation
3. `normalizeFileEditInput()` - input normalization
4. `validateInput()` - tool-specific validation
5. **PreToolUse Hook** (parallel with speculative classifier)
6. BashTool safety classifier
7. `checkPermissions()` - allow/ask/deny
8. User confirmation (if ask)
9. `call()` - core execution with onProgress
10. **PostToolUse Hook**
11. `mapToolResultToToolResultBlockParam()` - serialize
12. Large result persistence (>50K chars to `~/.claude/tool-results/`)

### Concurrency Control
- `StreamingToolExecutor` starts tools during streaming (before API response completes)
- `isConcurrencySafe(input)` is a **function**, not boolean (BashTool dynamically checks read-only commands)
- `MAX_TOOL_USE_CONCURRENCY = 10`
- Bash error triggers sibling abort (only Bash, not other tools)
- Tool states: `queued -> executing -> completed -> yielded`

### Result Size Limits
| Level | Constant | Value |
|-------|----------|-------|
| Single tool | `DEFAULT_MAX_RESULT_SIZE_CHARS` | 50,000 |
| Absolute cap | `MAX_TOOL_RESULT_TOKENS` | 100,000 (~400KB) |
| Message aggregate | `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS` | 200,000 |

---

## 3. Subagents / Multi-Agent

### Three Progressive Modes
| Mode | Gate | Description |
|------|------|-------------|
| Sub-Agent | Default, via AgentTool | Single child agent for specific tasks |
| Coordinator-Worker | `COORDINATOR_MODE` feature + env var | Main thread only dispatches; workers do implementation |
| Swarm Team | `isAgentSwarmsEnabled()` triple gate | Multiple teammates with mailbox communication + shared task list |

### AgentTool Dispatch (`AgentTool.tsx`, 1,834 lines)
- **Fork path**: `effectiveType === undefined`, builds cache-consistent prefix via `buildForkedMessages()`, unified async
- **Normal path**: explicit `subagent_type`, `resolveAgentTools()` filtering
- Fork maximizes prompt cache hit rate: inherits parent's rendered system prompt, uses exact tools, placeholder tool_results

### Agent Execution (`runAgent.ts`, 973 lines)
- `async function*` generator, uses same `query()` as main agent
- Hooks registration via `registerFrontmatterHooks()`
- Skills loading from agent frontmatter
- MCP server initialization per agent
- Context isolation via `createSubagentContext()` (independent messages, readFileState, abortController)

### Built-in Agent Types
| Type | Purpose | Special Config |
|------|---------|----------------|
| General Purpose | Default sub-agent | - |
| Explore | Read-only search | `omitClaudeMd: true`, haiku model for external |
| Plan | Read-only planning | `omitClaudeMd: true` |
| Verification | Verify changes | Disallows Agent/Edit/Write/NotebookEdit |
| Fork | Cache-optimized | `permissionMode: 'bubble'`, `model: 'inherit'` |

### Coordinator Mode
- Tools restricted to: AgentTool, TaskStopTool, SendMessageTool, SyntheticOutputTool
- Four-phase workflow: Research -> Synthesis -> Implementation -> Verification
- System prompt (~258 lines) with decision tables for Continue vs Spawn

### Swarm Team Mode
- **Mailbox system**: `~/.claude/teams/{team}/inboxes/{agent}.json`, file-level locks via `proper-lockfile`, 10 retries with 5-100ms exponential backoff
- **TodoV2 tasks**: per-task JSON files, two-level locking (task-level + directory-level), 30 retries for ~10+ concurrent agents
- **Worker backends**: tmux (internal/external), iTerm2 (native panels), InProcess (AsyncLocalStorage)

### Task Types (7)
`local_bash`, `local_agent`, `remote_agent`, `in_process_teammate`, `local_workflow`, `monitor_mcp`, `dream`

### Task Status Flow
`pending -> running -> completed/failed/killed`

---

## 4. Skills

### Design Philosophy
"Prompt as Capability" - SKILL.md file = discoverable, invocable, composable capability unit.

### Discovery Sources (5 layers, high to low priority)
1. Managed policy layer (`getManagedFilePath()/.claude/skills/`)
2. User layer (`~/.claude/skills/`)
3. Project layer (upward traversal of `.claude/skills/`)
4. Bundled skills (`src/skills/bundled/`)
5. MCP skills (`appState.mcp.commands`)

### Loading (3 phases)
1. **Startup scan**: directory traversal, frontmatter parse, realpath dedup
2. **Dynamic discovery**: `discoverSkillDirsForPaths()` - upward traversal on file operations
3. **Conditional activation**: `paths` field matching for activation

### Frontmatter Fields
`name`, `description`, `when_to_use`, `version`, `user-invocable`, `disable-model-invocation`, `paths`, `context` (`'fork'`), `agent`, `model`, `effort`, `allowed-tools`, `hooks`, `shell`

### Execution (SkillTool)
- **Inline mode**: `processPromptSlashCommand()`, modifies current context via `contextModifier`
- **Fork mode**: `executeForkedSkill()` -> `runAgent()`, isolated session
- Token budget: 1% of context window (`SKILL_BUDGET_CONTEXT_PERCENT = 0.01`)
- Usage scoring: `score = usageCount * max(0.5^(daysSinceUse/7), 0.1)`

### Built-in Skills (~15)
`/update-config`, `/keybindings`, `/verify`, `/debug`, `/lorem-ipsum`, `/skillify`, `/remember`, `/simplify`, `/batch`, `/stuck`, `/loop`, `/cron-list`, `/cron-delete`, `/dream`, plus feature-gated ones

---

## 5. Context Management

### Window Specifications
| Spec | Capacity |
|------|----------|
| Standard | 200,000 tokens (`MODEL_CONTEXT_WINDOW_DEFAULT`) |
| Large context | 1,000,000 tokens (model ID contains `[1m]`) |

### Effective Window
```
effective = total - Math.min(model_output_limit, 20,000)
```

### Threshold Layers
| Threshold | Formula |
|-----------|---------|
| Auto-compact | effective - 13K (`AUTOCOMPACT_BUFFER_TOKENS`) |
| Warning | auto-compact - 20K |
| Error | auto-compact - 20K |
| Blocking limit | effective - 3K |

### Token Counting: `tokenCountWithEstimation()`
- Hybrid algorithm: API baseline (last assistant message's usage) + heuristic estimation for subsequent messages
- Handles parallel tool call message splitting (same message.id detection)
- Fallback to pure heuristic when no API usage exists

### System Prompt Architecture
- **7 static sections** (cached, `cacheScope: 'global'`):
  1. Intro, 2. System, 3. Doing Tasks, 4. Actions, 5. Using Your Tools, 6. Tone and Style, 7. Output Efficiency
- **`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`** marker separates static from dynamic
- **Dynamic sections**: session_guidance, memory (memdir), env_info, language, output_style, mcp_instructions (the only `DANGEROUS_uncached` section by default), plus feature-gated sections

### Prompt Cache Economics
- Boundary before: global cache scope (same across all users)
- Section-level in-memory memoize
- `DANGEROUS_uncachedSystemPromptSection` forces recalculation each turn
- Cache invalidation: `/compact`, session resume, worktree switch

### Compression Pipeline (4 levels)
1. **MicroCompact** (`microCompact.ts`, ~530 lines) - local tool result replacement with `'[Old tool result content cleared]'`, no API call
2. **Session Memory Compact** (`sessionMemoryCompact.ts`, ~630 lines) - structured trimming, preserves 10K-40K tokens, no API call, no custom instructions support
3. **Full Compact** (`compact.ts`, ~1,708 lines) - API summary with 9-section structured template, max_tokens=20K, PTL emergency fallback
4. **Reactive Compact** - feature-gated stub

### Post-Compact Recovery
- File attachments: 50K token budget, 5K/file, max 5 files
- Skills: 25K token budget, 5K/skill
- Plan attachments, tool/agent increments, MCP instruction increments

### ContentReplacementState (Frozen Decision)
- `seenIds: Set<string>` - model has seen these tool_use IDs
- `replacements: Map<string, string>` - already replaced IDs
- Three partitions: mustReapply, frozen, fresh
- Ensures prompt cache prefix stability

---

## 6. Permission System

### Three Verdicts
**Allow** (auto-pass), **Ask** (interactive confirmation), **Deny** (reject with reason), plus internal **Passthrough**

### Permission Check Pipeline (`hasPermissionsToUseToolInner`)
1. Tool-level Deny rules
2. Tool-level Ask rules (with sandbox exception for Bash)
3. `tool.checkPermissions()` - tool's own logic
4. Tool Deny (bypass-immune)
5. `requiresUserInteraction` Ask (bypass-immune)
6. Content-level Ask rules (bypass-immune)
7. Safety path checks (bypass-immune) - `.git/`, `.claude/`, shell configs
8. Bypass mode check
9. Always-Allow rules
10. Passthrough -> Ask (default)

### Seven Permission Modes
| Mode | Type | Behavior |
|------|------|----------|
| `default` | External | Unmatched operations need user confirmation |
| `acceptEdits` | External | File edits in working dir auto-allowed |
| `plan` | External | Read-only via system prompt instruction |
| `dontAsk` | External | Ask -> Deny (unattended scenarios) |
| `bypassPermissions` | External | Bypass most checks (except bypass-immune) |
| `auto` | Internal | AI classifier decides |
| `bubble` | Internal | Sub-agent bubbles to parent |

### Auto Mode: Dual-Stage XML Classifier
- **3-tier fast paths** before classifier:
  1. acceptEdits fast path (re-run checkPermissions in acceptEdits mode)
  2. Safe tool whitelist (`SAFE_YOLO_ALLOWLISTED_TOOLS`)
  3. Dual-stage XML classifier (via `sideQuery`)
- **Stage 1**: Quick decision, max_tokens=64, stop on `</block>`, `<block>no</block>` = allow
- **Stage 2**: Deep thinking, max_tokens=4096, `<thinking>` reasoning + `<block>` decision
- Fail-closed: unparseable = block
- Excludes assistant text from transcript (anti-social-engineering)
- Denial tracking: consecutive >= 3 or total >= 20 -> fallback to user prompt

### Rule Sources (8)
`session`, `cliArg`, `command`, `userSettings`, `projectSettings`, `localSettings`, `flagSettings`, `policySettings`

### Rule Matching
- Three patterns: exact, prefix (`:*` suffix), wildcard (`*`)
- Shell commands: tree-sitter AST parsing, 23 static security checks
- 30+ safe env vars whitelisted, dangerous ones (PATH, LD_PRELOAD) participate in matching

### Sandbox
- macOS: `sandbox-exec` (system built-in)
- Linux: bubblewrap (`bwrap` + `socat`)
- Three isolation dimensions: filesystem (allowWrite/denyWrite/allowRead/denyRead), network (domain whitelist), command exclusion

---

## 7. MCP Integration

### Five-Stage Pipeline
1. **Config merge**: 7 scopes (dynamic > local > project > user > enterprise > claudeai > managed), `Object.assign` override
2. **Transport establishment**: `connectToServer()` with memoize cache
3. **Tool discovery**: `fetchToolsForClient()` via `tools/list`, LRU cache (size 20)
4. **Name construction**: `mcp__${serverName}__${toolName}`, character normalization
5. **Pool merge**: `assembleToolPool()` with partition-sort + uniqBy dedup

### Eight Transport Types
`stdio`, `sse`, `http`, `sse-ide`, `ws-ide`, `ws`, `claudeai-proxy`, `sdk`

### Connection Management
- 5 connection states: `connected`, `pending`, `failed`, `needs-auth`, `disabled`
- Terminal error detection (ECONNRESET, ETIMEDOUT, etc.), 3 consecutive -> reconnect
- Session expiry: HTTP 404 + JSON-RPC -32001
- Real-time sync: ToolListChanged, PromptListChanged, ResourceListChanged notifications
- 16ms batch flush for state updates

### Tool Annotation Mapping
- `readOnlyHint` -> `isReadOnly()` / `isConcurrencySafe()`
- `destructiveHint` -> `isDestructive()`
- `openWorldHint` -> `isOpenWorld()`
- All MCP tools: `checkPermissions()` returns `passthrough` (must get user confirmation)

### Enterprise Exclusion
When enterprise config exists, **all other scopes are excluded entirely** (not just overridden).

---

## 8. Configuration

### Five-Layer Settings Merge (later overrides earlier)
1. `userSettings` - `~/.claude/settings.json`
2. `projectSettings` - `.claude/settings.json`
3. `localSettings` - `.claude/settings.local.json`
4. `flagSettings` - CLI `--settings` parameter
5. `policySettings` - `managed-settings.d/` (systemd-style drop-in directory)

### Runtime Config
- `GlobalConfig`: `~/.claude.json` (with reentrant guard for error handling)
- `ProjectConfig`: `.claude/config.json`

### CLAUDE.md System
- `src/utils/claudemd.ts` (~1,480 lines)
- Hierarchical discovery: managed (`/etc/claude-code/CLAUDE.md`) > user (`~/.claude/CLAUDE.md`) > project (`{root}/CLAUDE.md`) > local (`CLAUDE.local.md`) > memory index (`memory/MEMORY.md`)
- `@include` directive system: `@path`, `@./relative`, `@~/home`, `@/absolute`
- Leaf text node extraction via `marked` Lexer (skips code blocks)
- Circular reference detection + `MAX_INCLUDE_DEPTH = 5`
- ~100 text file extensions whitelisted

### Zod v4 Schema Validation
- `lazySchema()` factory (8 lines) for deferred schema construction
- Settings validated at runtime

---

## 9. Session/Message Model

### JSONL Transcript Storage
- Path: `~/.claude/projects/{sanitized-cwd}/{session-id}.jsonl`
- Sub-agent transcripts: `{sessionDir}/subagents/agent-{id}.jsonl`
- Append-only with `0o600` file permissions
- Async queue: `enqueueWrite()` -> `scheduleDrain()` -> 100ms batch merge
- Sync direct write: `appendFileSync()` for exit/compact/resume

### Entry Types (18+)
`TranscriptMessage`, `SummaryMessage`, `CustomTitleMessage`, `AiTitleMessage`, `LastPromptMessage`, `TaskSummaryMessage`, `TagMessage`, `AgentNameMessage`, `AgentColorMessage`, `AgentSettingMessage`, `PRLinkMessage`, `ModeEntry`, `WorktreeStateEntry`, `ContentReplacementEntry`, `FileHistorySnapshotMessage`, `AttributionSnapshotMessage`, `QueueOperationMessage`, `SpeculationAcceptMessage`, `ContextCollapseCommitEntry`, `ContextCollapseSnapshotEntry`

### parentUuid Chain
- Single-linked list: each message's `parentUuid` points to previous
- Fork branches: sub-agent transcripts inherit parent UUID
- Compact boundary: `parentUuid = null` marks new chain start
- Chain building: `buildConversationChain()` - leaf-to-root traversal + reverse
- Cycle detection via `seen: Set<UUID>`

### Session Recovery
1. `readTranscriptForLoad()` - 1MB chunk reads, skip attribution-snapshot
2. `walkChainBeforeParse()` - byte-level chain traversal (discard unrelated forks)
3. `progressBridge()` - legacy progress message bridging
4. `buildConversationChain()` - leaf->root + reverse
5. Recovery: `filterUnresolvedToolUses()`, `detectTurnInterruption()`
6. State rebuild

### 64KB Lite Metadata Window
- Head 64KB: `isSidechain`, `projectPath`, `teamName`, `gitBranch`
- Tail 64KB: `customTitle`, `aiTitle`, `summary`, `tag`, `lastPrompt`, PR info
- `reAppendSessionMetadata()` ensures metadata stays in tail window

---

## 10. Server/API

### Multi-Provider Architecture (6 providers)
| Provider | SDK | Auth |
|----------|-----|------|
| firstParty (Anthropic) | `Anthropic` | API Key / OAuth |
| bedrock | `AnthropicBedrock` | AWS credentials |
| vertex | `AnthropicVertex` | Google Cloud credentials |
| foundry | `AnthropicFoundry` | Azure AD / API Key |
| openai | OpenAI SDK | `OPENAI_API_KEY` |
| gemini | Raw fetch | `GEMINI_API_KEY` |

### Request Flow
```
queryModelWithStreaming() -> withStreamingVCR() -> queryModel()
  -> queryModelOpenAI() (OpenAI path)
  -> queryModelGemini() (Gemini path)
  -> for await (stream) (Anthropic streaming loop)
```

### SSE Event Processing
- Direct consumption of `BetaRawMessageStreamEvent` (not SDK high-level wrapper)
- Events: `message_start` -> `content_block_start/delta/stop` -> `message_delta` -> `message_stop`
- JSON tool parameter fragments concatenated as strings during streaming, parsed only at `content_block_stop`
- Direct property mutation for usage data (preserves transcript queue references)

### Retry Strategy (`withRetry.ts`, 822 lines)
- 429: exponential backoff with retry-after header
- 529: max 3 retries (foreground), immediate fail (background), fallback model trigger
- 401: force OAuth refresh
- 400 context overflow: reduce max_tokens
- ECONNRESET/EPIPE: disable keep-alive
- Persistent retry mode: 5min max backoff, 6h total cap, 30s heartbeat

### Stream Monitoring
- Idle watchdog: configurable timeout (default 90s), fallback to non-streaming
- Stall detection: 30s threshold, retrospective metrics
- Resource release: timers, response body, SDK stream resources

### SDK / Headless Mode
- `runHeadless()` for `-p`/`--print` or non-TTY
- Independent `headlessStore` (Zustand) replaces Ink state management
- SDK mode: auto sets `stream-json` format, enables verbose

---

## 11. UI

### Terminal React Application
- **React 19** with ConcurrentRoot
- **Custom Reconciler** (`src/ink/reconciler.ts`, 512 lines) - deep fork of Ink framework
- **104 files** in `src/ink/`, ~647KB
- **Yoga layout engine** for terminal flexbox
- **8 node types**: `ink-root`, `ink-box`, `ink-text`, `ink-virtual-text`, `ink-link`, `ink-progress`, `ink-raw-ansi`, `#text`

### Int32Array Double-Buffer Screen (`screen.ts`, 1,486 lines)
- Each cell: 2 Int32Array slots (word0: charId, word1: packed styleId/hyperlinkId/width)
- BigInt64Array overlay for batch zero (`cells64.fill(0n)`)
- Three object pools: CharPool (ASCII fast path), HyperlinkPool, StylePool
- Frame diff: compare prev/next cells, output only changed ANSI sequences
- 16ms frame interval (~60fps)

### Key Components
- `REPL.tsx` (6,182 lines) - main interface
- `StreamingMarkdown` - stable prefix optimization, plain text fast path
- Virtual scroll: `DEFAULT_ESTIMATE=3`, `OVERSCAN_ROWS=80`, `MAX_MOUNTED_ITEMS=300`
- Vim mode: 2 modes (INSERT/NORMAL), 11 CommandState variants, operators x motions x text objects

### State Management
- **34-line Store** (`store.ts`): `Object.is` short-circuit, `onChange` callback
- No Zustand/Redux - custom implementation
- `DeepImmutable<>` wrapper for AppState
- `onChangeAppState` - single-throat pattern for all state changes
- 5 React Contexts: AppContext, TerminalFocusContext, TerminalSizeContext, StdinContext, ClockContext

### W3C Event System
- Three-phase dispatch: capture -> target -> bubble
- `DiscreteEventPriority` vs `ContinuousEventPriority`
- Focus management: stack-based, max 32 entries

---

## 12. Hooks System

### 27 Event Types
```
PreToolUse, PostToolUse, PostToolUseFailure, Notification, UserPromptSubmit,
SessionStart, SessionEnd, Stop, StopFailure, SubagentStart, SubagentStop,
PreCompact, PostCompact, PermissionRequest, PermissionDenied, Setup,
TeammateIdle, TaskCreated, TaskCompleted, Elicitation, ElicitationResult,
ConfigChange, WorktreeCreate, WorktreeRemove, InstructionsLoaded, CwdChanged, FileChanged
```

### Six Execution Types
| Type | Timeout | Method |
|------|---------|--------|
| `command` | 10 min | Shell subprocess, stdin JSON, stdout result |
| `prompt` | 30 sec | LLM call (default Haiku) |
| `http` | 10 min | POST with 5-layer security |
| `agent` | 60 sec | Sub-agent verifier |
| `callback` | - | SDK internal |
| `function` | - | Session-scoped internal |

### Three Configuration Channels
1. **Settings snapshot** (frozen at startup, from 5 settings sources)
2. **Registered hooks** (runtime SDK callbacks + plugin hooks)
3. **Session hooks** (frontmatter hooks from Skills/Agents, isolated by sessionId)

### Seven-Stage Execution Pipeline
0. `hasHookForEvent()` - quick existence check
1. `shouldSkipHookDueToTrust()` - workspace trust gate
2. `getMatchingHooks()` - matcher + if-condition double filter
3. `hookDedupKey()` dedup
4. Parallel execution
5. `parseHookOutput()` - JSON or plain text
6. Result aggregation (deny-wins, context merge)

### Hook Permission Integration
Priority: Hook deny > Settings deny > Settings ask > Hook allow > Normal flow

---

## 13. Tech Stack

| Area | Choice | Purpose |
|------|--------|---------|
| Runtime | **Bun** (>= 1.3) | JS runtime, bundler, test framework |
| Language | **TypeScript** | Type safety |
| UI | **React 19 + Ink** (deep fork) | Terminal declarative UI, Yoga layout |
| CLI | **Commander.js** | Command-line parsing |
| API SDK | **@anthropic-ai/sdk** | Anthropic API streaming |
| Schema | **Zod v4** | Runtime type validation |
| MCP | **@modelcontextprotocol/sdk** | MCP server integration |
| Lint | **Biome** | Formatting and static analysis |
| Package | **Bun Workspaces** | Monorepo management |
| Build | Custom `build.ts` | Code splitting, Node.js/Bun dual-runtime output |
| Parsing | **tree-sitter** | Bash command AST analysis |
| State | Custom 34-line store | `Object.is` short-circuit, onChange callback |

### Dependency Injection
- `QueryEngine` receives all dependencies via config object (tools, commands, mcpClients)
- `query()` receives `deps` with `callModel`, `checkPermissions`, etc.
- No DI framework - manual constructor injection

### Feature Flag System (3 layers)
1. **Compile-time DCE**: `import { feature } from 'bun:bundle'` - Bun tree-shakes disabled branches
2. **Runtime env/identity**: `FEATURE_<NAME>=1`, `USER_TYPE === 'ant'`
3. **GrowthBook remote**: `tengu_*` prefix flags, server-side evaluation, disk cache

---

## 14. Design Philosophy

### Seven Core Principles
1. **Don't trust model self-awareness**: All behavior constraints explicitly in prompt (900+ line system prompt, per-tool prompt files)
2. **Separate roles**: Specialized agents (Explore, Plan, Verification) reduce cognitive interference
3. **Govern tool calls**: 14-step pipeline where 13 steps are governance
4. **Context is budget**: Finite resource with allocation optimization (static/dynamic partitioning, 4-level compression, token estimation at 3 precision levels)
5. **Security layers don't overlap**: Defense in depth with strict responsibility boundaries (Hook allow doesn't bypass rule deny)
6. **Ecosystem needs model awareness**: Registration + injection (MCP instructions, skill discovery, agent list in prompt)
7. **Productionization is about day two**: Session recovery, error recovery matrix, agent cleanup chains, resource lifecycle

### Key Engineering Patterns
- **Leaf Module**: `bootstrap/state.ts` (1,758 lines) - imported by everyone, imports nothing from project (ESLint enforced)
- **Frozen Decision (Latch)**: 4 three-state fields (`null -> true -> stays true`) protecting prompt cache headers
- **AsyncGenerator Pipeline**: `REPL -> QueryEngine -> query() -> queryModelWithStreaming()` with backpressure
- **Observable Autonomy**: 14-step tool pipeline + StreamingToolExecutor with real-time progress
- **Sectioned Cache**: Static/dynamic prompt partitioning + beta header latching + fork inheritance

### Memory System
- **Auto-memory (memdir)**: 4 types (user, feedback, project, reference), Sonnet-driven semantic retrieval (top-5), background forked agent extraction
- **Session Memory**: Per-session markdown summary, 10K token init threshold, 5K between updates
- **CLAUDE.md hierarchy**: managed > user > project > local > memory index

### Bridge/Remote Control
- 3 generations: WebSocket+POST -> SSE+CCR -> Env-Less direct connect
- Up to 32 concurrent sessions
- Pointer-based crash recovery with 4h TTL

### Computer Use
- 38 MCP tools (24 universal + 11 Windows + 3 teaching)
- Platform backends: macOS (AppleScript/JXA), Windows (SendInput/UI Automation/COM), Linux (xdotool/scrot/xrandr)

### Voice Mode
- Push-to-talk via WebSocket STT (`voice_stream` endpoint)
- OAuth-only (not available for API key users)
- Deepgram STT (optional Nova 3 via GrowthBook gate)
