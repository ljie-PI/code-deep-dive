# 附录 A：关键文件索引

> 按行数降序排列，列出 `packages/opencode/src/` 中最重要的 50 个文件。

| 排名 | 文件路径 | 行数 | 章节 | 职责 |
|------|---------|------|------|------|
| 1 | `cli/cmd/tui/routes/session/index.tsx` | 2288 | Ch.12 | TUI 会话视图 — 消息渲染、工具展示 |
| 2 | `lsp/server.ts` | 1958 | Ch.15 | LSP 服务器实现 |
| 3 | `session/prompt.ts` | 1859 | Ch.2 | **核心循环** — Agent loop、工具解析、消息组装 |
| 4 | `acp/agent.ts` | 1842 | Ch.15 | ACP Agent 实现 |
| 5 | `provider/sdk/copilot/responses/openai-responses-language-model.ts` | 1769 | Ch.4 | Copilot Responses API SDK |
| 6 | `provider/provider.ts` | 1709 | Ch.4 | Provider 核心 — 工厂、模型解析、20+ 提供商 |
| 7 | `config/config.ts` | 1661 | Ch.8 | 配置系统 — Schema、多层合并、读写 |
| 8 | `cli/cmd/github.ts` | 1646 | Ch.12 | GitHub CLI 集成 |
| 9 | `cli/cmd/tui/component/prompt/index.tsx` | 1274 | Ch.12 | TUI 输入组件 — 自动补全、文件引用 |
| 10 | `cli/cmd/tui/context/theme.tsx` | 1238 | Ch.12 | TUI 主题系统 |
| 11 | `server/instance/session.ts` | 1110 | Ch.11 | Session API 路由 |
| 12 | `provider/transform.ts` | 1067 | Ch.4 | Provider 格式转换层 |
| 13 | `session/message-v2.ts` | 1057 | Ch.3 | 消息模型 v2 — Part 类型、格式转换 |
| 14 | `cli/cmd/tui/plugin/runtime.ts` | 1031 | Ch.9 | TUI 插件运行时 |
| 15 | `mcp/index.ts` | 930 | Ch.15 | MCP 客户端 — 连接管理、工具获取 |
| 16 | `cli/cmd/tui/app.tsx` | 864 | Ch.12 | TUI 应用根组件 |
| 17 | `session/index.ts` | 818 | Ch.3 | Session CRUD 操作 |
| 18 | `provider/sdk/copilot/chat/openai-compatible-chat-language-model.ts` | 815 | Ch.4 | Copilot Chat API SDK |
| 19 | `cli/cmd/mcp.ts` | 796 | Ch.12 | MCP 管理 CLI 命令 |
| 20 | `snapshot/index.ts` | 779 | Ch.13 | 快照系统 — track/restore/revert/diff |
| 21 | `cli/cmd/tui/routes/session/permission.tsx` | 691 | Ch.12 | TUI 权限确认对话框 |
| 22 | `cli/cmd/run.ts` | 691 | Ch.12 | 默认命令 — TUI 启动、工具展示 |
| 23 | `tool/edit.ts` | 688 | Ch.6 | 编辑工具（精确字符串替换） |
| 24 | `patch/index.ts` | 680 | Ch.6 | 补丁应用 |
| 25 | `file/index.ts` | 678 | — | 文件操作服务 |
| 26 | `cli/cmd/tui/component/prompt/autocomplete.tsx` | 672 | Ch.12 | TUI 自动补全 |
| 27 | `cli/cmd/tui/component/logo.tsx` | 633 | Ch.12 | TUI Logo 渲染 |
| 28 | `session/processor.ts` | 617 | Ch.2 | LLM 流式调用处理 |
| 29 | `plugin/codex.ts` | 608 | Ch.9 | Codex 插件 |
| 30 | `worktree/index.ts` | 600 | Ch.13 | Git worktree 管理 |
| 31 | `file/ripgrep.ts` | 589 | — | ripgrep 集成 |
| 32 | `lsp/index.ts` | 537 | Ch.15 | LSP Service |
| 33 | `effect/cross-spawn-spawner.ts` | 514 | Ch.10 | 跨平台进程生成 |
| 34 | `tool/bash.ts` | 510 | Ch.6 | Bash 工具实现 |
| 35 | `project/project.ts` | 487 | — | 项目管理 |
| 36 | `account/index.ts` | 456 | — | 账户管理 |
| 37 | `session/llm.ts` | 452 | Ch.2 | LLM 抽象层 |
| 38 | `v2/session-event.ts` | 443 | — | v2 会话事件 |
| 39 | `plugin/install.ts` | 439 | Ch.9 | 插件安装 |
| 40 | `storage/json-migration.ts` | 425 | Ch.17 | JSON → SQLite 迁移 |
| 41 | `server/instance/experimental.ts` | 425 | Ch.11 | 实验性 API 路由 |
| 42 | `agent/agent.ts` | 412 | Ch.5 | Agent 配置与生成 |
| 43 | `session/compaction.ts` | 409 | Ch.14 | 上下文压缩 |
| 44 | `cli/cmd/tui/context/local.tsx` | 412 | Ch.12 | TUI 本地状态 |
| 45 | `cli/cmd/stats.ts` | 413 | Ch.12 | 统计信息 CLI |

## 规模统计

| 指标 | 值 |
|------|-----|
| 核心包总行数 | 77,441 |
| TypeScript 文件总数 | ~350 (核心包), ~1421 (全 monorepo) |
| 子目录数 | ~40 |
| 最大文件 | 2,288 行 (TUI SessionRoute) |
| 中位数文件 | ~100 行 |

## 文件类型分布

| 类型 | 数量 | 说明 |
|------|------|------|
| `.ts` | ~300 | TypeScript 逻辑 |
| `.tsx` | ~50 | SolidJS 组件 (TUI) |
| `.txt` | ~15 | 系统提示模板 |
| `.sql.ts` | ~5 | Drizzle ORM 表定义 |
