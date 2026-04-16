# 附录 B：术语表

| 术语 | 定义 |
|------|------|
| **Agent** | 一组配置的组合（模型 + 工具 + 权限 + 提示），定义了 AI 助手的行为模式 |
| **ACP** | Agent Client Protocol — 让 OpenCode 作为可被编排的 Agent |
| **Bus** | 事件总线 — 基于 Effect PubSub 的发布/订阅系统 |
| **Compaction** | 上下文压缩 — 将旧消息替换为摘要以节省令牌 |
| **Effect** | TypeScript 函数式编程库，用于依赖注入、并发控制和错误处理 |
| **Instance** | 项目实例 — 每个工作目录对应一个独立的运行时上下文 |
| **InstanceState** | per-directory 可变状态管理，确保多项目间状态隔离 |
| **Layer** | Effect 的依赖实现单元，声明服务的具体实现和它的依赖 |
| **LLM** | Large Language Model — 大语言模型 |
| **MCP** | Model Context Protocol — 连接外部工具服务器的开放标准 |
| **LSP** | Language Server Protocol — 代码诊断和智能感知协议 |
| **MessageV2** | 消息模型 v2 — 支持多种 Part 类型的消息系统 |
| **Namespace** | TypeScript namespace 模式 — OpenCode 用来组织模块的核心模式 |
| **opentui** | SolidJS 终端渲染引擎 — 将 SolidJS 组件渲染到终端 |
| **Part** | 消息的内容单元 — text、tool、file、subtask、compaction 等类型 |
| **Permission** | 权限系统 — allow/deny/ask 三种动作，wildcard 规则匹配 |
| **Plugin** | 插件 — npm 包或本地模块，扩展 OpenCode 的工具、Agent、命令等 |
| **Projector** | 投射器 — 将内部事件转换为外部 SSE/WebSocket 事件 |
| **Provider** | LLM 提供商 — Anthropic、OpenAI、Google 等 AI 服务 |
| **ProviderTransform** | 提供商转换层 — 处理不同 LLM 的格式差异 |
| **PTY** | Pseudo-Terminal — 伪终端，Bash 工具通过它执行命令 |
| **Ruleset** | 规则集 — Permission.Rule 的数组，按 last-match-wins 求值 |
| **SDK** | Software Development Kit — 自动生成的 TypeScript 客户端 |
| **Service** | Effect Context Tag — 声明一个依赖的接口标识 |
| **Session** | 会话 — 用户与 Agent 的一次交互，包含消息历史 |
| **Skill** | 技能 — SKILL.md 文件定义的可执行指令模板 |
| **Slug** | URL 友好标识 — 每个 Session 的短标识（如 "happy-tiger"） |
| **Snapshot** | 快照 — 独立 Git 仓库中的文件状态记录 |
| **ULID** | Universally Unique Lexicographically Sortable Identifier — 时间有序 ID |
| **Variant** | 变体 — Agent 在不同上下文下的配置版本 |
| **Worktree** | 工作树 — Git worktree，支持在同一仓库的多个工作副本 |
