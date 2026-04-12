# 附录 B：术语表

| 术语 | 定义 |
|------|------|
| **Archon** | 项目名称。AI 编码助手的工作流引擎 |
| **Branded Type** | TypeScript 技术，通过 `unique symbol` 使 `string` 类型在编译期互不兼容（如 `RepoPath`、`BranchName`） |
| **CLIAdapter** | `IPlatformAdapter` 的命令行实现，输出到 stdout |
| **Codebase** | 在 Archon 中注册的 Git 仓库，存储在 `remote_agent_codebases` 表 |
| **Command** | `.archon/commands/` 中的 Markdown 文件，作为 AI 执行的提示模板 |
| **Conversation** | 一次完整的人机对话，由 `(platform_type, platform_conversation_id)` 唯一标识 |
| **ConversationLockManager** | 确保同一会话的消息串行处理的并发控制器 |
| **DAG** | Directed Acyclic Graph（有向无环图），工作流的执行模型 |
| **DagNode** | 工作流 DAG 中的一个节点，7 种类型之一：command、prompt、bash、loop、approval、cancel、script |
| **Env Leak Gate** | 安全机制，防止目标仓库的 `.env` 中的敏感变量泄漏到 AI subprocess |
| **handleMessage()** | 系统的中央入口函数，所有平台消息都流经此处 |
| **IAssistantClient** | AI 客户端接口，`ClaudeClient` 和 `CodexClient` 实现此接口 |
| **IDatabase** | 数据库抽象接口，同时支持 SQLite 和 PostgreSQL |
| **IPlatformAdapter** | 平台适配器接口，所有通信平台实现此接口 |
| **IIsolationProvider** | 隔离提供者接口，`WorktreeProvider` 是唯一实现 |
| **IIsolationStore** | 隔离数据存储接口，由 `@archon/core` 注入到 `@archon/isolation` |
| **IsolationBlockedError** | 特殊错误类型，表示隔离创建失败且用户已被通知，所有后续处理必须停止 |
| **IsolationHints** | 轻量级隔离上下文，由平台适配器提供给编排器 |
| **IsolationResolver** | 7 步隔离解析引擎，决定使用已有环境还是创建新环境 |
| **IWorkflowStore** | 工作流存储接口（`@archon/workflows` 中定义），由 `@archon/core` 实现 |
| **MessageChunk** | AI 响应的流式块，判别联合类型（assistant、tool、result 等） |
| **MergedConfig** | 全局配置和仓库配置合并后的结果 |
| **Session** | AI SDK 会话，不可变——状态变化通过创建新 Session 实现 |
| **SqlDialect** | 数据库方言抽象，处理 PostgreSQL 和 SQLite 的语法差异 |
| **SSETransport** | Server-Sent Events 连接管理器，Web UI 的实时通信基础 |
| **stripCwdEnv()** | 启动时清洗 Bun 自动加载的 CWD `.env` 变量和 Claude Code 嵌套标记 |
| **TransitionTrigger** | Session 状态转换的触发器（如 `first-message`、`plan-to-execute`、`reset-requested`） |
| **TriggerRule** | DAG 节点的依赖合并策略（`all_success`、`one_success`、`none_failed_min_one_success`、`all_done`） |
| **WebAdapter** | `IWebPlatformAdapter` 实现，集成 SSE 推送、消息持久化和工作流事件桥 |
| **Workflow** | `.archon/workflows/` 中的 YAML 文件，定义多步骤 AI 任务的 DAG |
| **WorkflowDeps** | 工作流引擎的依赖注入接口，包含 store、AI 客户端工厂、配置加载器 |
| **WorkflowEventBridge** | 将工作流执行事件转发到 SSE 流的桥接器 |
| **Worktree** | Git worktree，Archon 的默认隔离机制——每个任务在独立的 worktree 中执行 |

## 环境变量速查

### 核心

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `ARCHON_HOME` | Archon 主目录 | `~/.archon` |
| `DATABASE_URL` | PostgreSQL 连接字符串 | 未设置→SQLite |
| `PORT` | HTTP 服务器端口 | `3090` |
| `LOG_LEVEL` | 日志级别 | `info` |
| `MAX_CONCURRENT_CONVERSATIONS` | 最大并发会话数 | `10` |

### AI 凭证

| 变量 | 说明 |
|------|------|
| `CLAUDE_USE_GLOBAL_AUTH` | 使用 `claude /login` 的全局认证 |
| `CLAUDE_API_KEY` | Claude API 密钥 |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code OAuth token |
| `CODEX_ID_TOKEN` + `CODEX_ACCESS_TOKEN` | Codex SDK 凭证 |
| `DEFAULT_AI_ASSISTANT` | 默认 AI 助手（`claude` / `codex`） |

### 平台

| 变量 | 平台 |
|------|------|
| `TELEGRAM_BOT_TOKEN` | Telegram |
| `DISCORD_BOT_TOKEN` | Discord |
| `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` | Slack |
| `GITHUB_TOKEN` + `WEBHOOK_SECRET` | GitHub |
| `GITEA_URL` + `GITEA_TOKEN` + `GITEA_WEBHOOK_SECRET` | Gitea |
| `GITLAB_TOKEN` + `GITLAB_WEBHOOK_SECRET` | GitLab |
