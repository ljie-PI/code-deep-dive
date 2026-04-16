# 1. 总览对比

## 1.1 基本信息

| 维度 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **定位** | Anthropic 官方 CLI 编码助手 | 开源 AI 编码 Agent | OpenAI 官方 CLI 编码助手 |
| **语言** | TypeScript (ESM) | TypeScript (ESM) | Rust (async tokio) |
| **运行时** | Bun >=1.2.0 | Bun | tokio async runtime |
| **代码规模** | ~2,895 .ts/.tsx 文件 | ~396 .ts/.tsx 文件 (核心包) | ~1,447 .rs 文件 (workspace) |
| **LLM SDK** | `@anthropic-ai/sdk` (自研多 Provider 适配) | Vercel AI SDK 6.x (15+ Provider 适配器) | 自研 `codex-api` crate (OpenAI Responses API) |
| **DI 方式** | 函数式组合 + `QueryDeps` 接口 | Effect 4.x Service/Layer 模式 | 结构体服务容器 (手动装配) |
| **UI 框架** | React 19 + Ink (自研 fork) | SolidJS + @opentui | ratatui + crossterm |
| **核心循环** | `async function*` generator | `while(true)` + Effect.gen | `loop {}` + tokio async |
| **数据存储** | JSONL 文件 (追加写入) | SQLite (Drizzle ORM) | JSONL + SQLite (双存储) |
| **架构风格** | 单体进程 | 微服务化 monorepo, Server-Client | 模块化 Rust workspace (~50 crates) |
| **Provider 支持** | 6 种 (Anthropic/Bedrock/Vertex/Foundry/OpenAI/Gemini) | 15+ 种 (通过 AI SDK 适配器) | OpenAI 为主 + 可配置 |
| **并发模型** | 单线程事件循环 + async generators | 单线程 + Effect fibers | 多线程 tokio (Arc/Mutex) |
| **包管理** | Bun (单包) | Bun workspaces (monorepo) | Cargo workspace (~50 crates) |

## 1.2 关键目录结构

### Claude Code (`src/`)
```
src/
├── services/api/       # LLM 通信 (claude.ts 3,455 行)
├── query.ts            # 核心 Agent 循环 (1,732 行)
├── services/tools/     # 工具编排
├── tools/              # 各工具实现
├── components/ + ink/  # React/Ink 终端 UI
├── utils/              # 大型工具层
├── bootstrap/          # 状态初始化
├── commands/           # Slash 命令
└── services/mcp/       # MCP 集成
```

### OpenCode (`packages/opencode/src/`)
```
packages/opencode/src/
├── session/            # 核心循环 (prompt.ts 1,859 行, processor.ts 617 行)
├── agent/              # Agent 定义
├── provider/           # 多 Provider 抽象
├── tool/               # 工具系统
├── storage/            # SQLite (Drizzle)
├── permission/         # 权限系统
├── config/             # 配置
├── server/             # Hono HTTP 服务器
└── bus/                # 事件总线
```

### Codex (`codex-rs/`)
```
codex-rs/
├── core/src/           # 核心逻辑 (codex.rs 8,211 行)
│   ├── agent/          # 多 Agent 控制
│   └── tools/          # 工具编排 + 沙箱
├── cli/                # CLI 入口
├── tui/                # ratatui 终端 UI
├── app-server/         # WebSocket 服务器
├── sandboxing/         # OS 级沙箱
├── codex-mcp/          # MCP 客户端
├── mcp-server/         # MCP 服务器
├── connectors/         # 外部集成
└── config/             # 配置 crate
```

## 1.3 技术栈对比

| 领域 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **构建** | Bun bundler (feature flag 编译时消除) | Bun (`script/build.ts`) | Cargo workspace + Bazel (CI) |
| **LLM 协议** | Anthropic Messages API (Beta) | Provider 无关 (AI SDK) | OpenAI Responses API (WebSocket + HTTPS) |
| **Schema** | Zod v4 | Zod + Effect Schema | serde + JSON Schema |
| **数据库** | JSONL 文件 | SQLite (bun:sqlite / Drizzle ORM) | JSONL + SQLite (`state_db`) |
| **HTTP** | 无 (CLI-only) | Hono 4.x | Axum (WebSocket) |
| **进程管理** | tmux / iTerm2 / AsyncLocalStorage | Effect ChildProcess | tokio tasks (Arc/Mutex) |
| **Lint** | Biome | Prettier | rustfmt / clippy |
