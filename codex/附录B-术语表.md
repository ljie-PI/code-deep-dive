# 附录B 术语表

> 格式：**术语** — 中文定义 — 来源文件/crate

---

## 核心抽象

| 术语 | 定义 | 来源 |
|------|------|------|
| **SQ (Submission Queue)** | 提交队列，用户/系统向 Agent 循环提交操作请求的通道 | `core/src/codex_thread.rs` |
| **EQ (Event Queue)** | 事件队列，Agent 循环向外部通知事件的通道 | `core/src/codex_thread.rs` |
| **Op** | 操作（Operation），SQ 中的一个提交单元，表示一个待处理的请求 | `core/src/codex_thread.rs` |
| **EventMsg** | 事件消息，EQ 中的一个通知单元，表示一个已发生的事件 | `app-server-protocol/src/protocol/common.rs` |
| **Submission** | 提交，封装了一个 Op 及其关联上下文的完整提交单元 | `core/src/codex_thread.rs` |
| **Event** | 事件，系统中发生的可观测状态变化的统称 | `app-server-protocol/src/protocol/common.rs` |

## 会话与线程

| 术语 | 定义 | 来源 |
|------|------|------|
| **Turn** | 轮次，一次用户输入到模型完成响应的完整交互周期 | `core/src/codex_thread.rs` |
| **Session** | 会话，从启动到关闭的完整 Codex 运行实例 | `rollout/src/lib.rs` |
| **Thread** | 线程，一个持久化的对话序列，由 ThreadId 唯一标识 | `thread-store/src/store.rs` |
| **ThreadId** | 线程标识符，全局唯一的线程句柄 | `codex-protocol` |
| **Rollout** | 卷出记录，会话事件的 JSONL 持久化文件 | `rollout/src/recorder.rs` |
| **RolloutItem** | 卷出条目，写入 JSONL 文件的单个事件记录 | `codex-protocol` |
| **RolloutRecorder** | 卷出录制器，负责将 RolloutItem 序列化并追加写入 JSONL 文件 | `rollout/src/recorder.rs` |
| **ThreadRecorder** | 线程录制器，存储无关的实时追加句柄，抽象了底层录制实现 | `thread-store/src/recorder.rs` |
| **ThreadStore** | 线程存储 trait，定义了线程 CRUD 的存储无关接口 | `thread-store/src/store.rs` |
| **LocalThreadStore** | 本地线程存储，ThreadStore 的文件系统实现 | `thread-store/src/local.rs` |
| **StoredThread** | 已存储线程，包含元数据和可选历史记录的线程摘要 | `thread-store/src/types.rs` |
| **ThreadPage** | 线程分页，包含线程列表和分页游标的查询结果 | `thread-store/src/types.rs` |
| **SessionSource** | 会话来源，标识线程的创建端（CLI、VSCode、atlas、chatgpt 等） | `codex-protocol` |

## MCP、Hook 与插件

| 术语 | 定义 | 来源 |
|------|------|------|
| **MCP (Model Context Protocol)** | 模型上下文协议，标准化的工具服务器通信协议 | `codex-mcp/src/lib.rs` |
| **Hook** | 钩子，在特定生命周期事件触发时执行的用户自定义脚本 | `core/src/hook_runtime.rs` |
| **Plugin** | 插件，通过 MCP 协议提供扩展功能的第三方包 | `core/src/plugins/manager.rs` |
| **Skill** | 技能，内置或可安装的专项能力模块（如 codex-pr-body） | `skills/src/lib.rs` |
| **Connector** | 连接器，第三方应用集成的发现与缓存机制 | `connectors/src/lib.rs` |

## 安全与权限

| 术语 | 定义 | 来源 |
|------|------|------|
| **SandboxPolicy** | 沙箱策略，定义文件系统和网络访问限制的安全边界 | `codex-protocol` |
| **AskForApproval** | 审批模式，控制工具调用是否需要用户确认的策略枚举 | `codex-protocol` |
| **ExecPolicy** | 执行策略，基于规则匹配的命令执行审批引擎 | `execpolicy/src/lib.rs` |
| **Guardian** | 守护者，基于 LLM 的自动审查系统，用于自动批准安全的工具调用 | `core/src/guardian/mod.rs` |
| **SecretsManager** | 密钥管理器，负责敏感信息的检测与保护 | `secrets/src/lib.rs` |
| **NetworkProxy** | 网络代理，拦截并控制沙箱内进程的网络请求 | `network-proxy/src/lib.rs` |
| **GhostCommit** | 幽灵提交，不影响当前分支的工作树快照提交，用于文件级撤销 | `git-utils/src/ghost_commits.rs` |

## 模型与 API

| 术语 | 定义 | 来源 |
|------|------|------|
| **ModelClient** | 模型客户端，向 LLM API 发送请求的抽象接口 | `core/src/client.rs` |
| **ModelClientSession** | 模型客户端会话，维护单个 API 会话的状态（如速率限制、重试） | `core/src/client.rs` |
| **Prompt** | 提示，发送给模型的完整请求，包含系统指令、历史消息和工具定义 | `core/src/context_manager/mod.rs` |
| **ResponseEvent** | 响应事件，从模型流式返回的单个事件（文本块、工具调用等） | `codex-api/src/sse/responses.rs` |
| **ModelsResponse** | 模型列表响应，包含可用模型信息的 API 响应 | `models-manager/src/manager.rs` |
| **AuthMode** | 认证模式，ApiKey（直接 API Key）或 Chatgpt（ChatGPT 认证流程） | `app-server-protocol/src/lib.rs` |

## 上下文管理

| 术语 | 定义 | 来源 |
|------|------|------|
| **ContextManager** | 上下文管理器，管理对话历史、工具结果和系统消息的上下文窗口 | `core/src/context_manager/mod.rs` |
| **Compaction** | 压缩，当上下文窗口接近上限时，通过 LLM 生成摘要来缩减历史消息 | `core/src/compact.rs` |
| **BaseInstructions** | 基础指令，会话级别的系统提示，持久化在线程元数据中 | `codex-protocol` |

## 工具系统

| 术语 | 定义 | 来源 |
|------|------|------|
| **ToolRouter** | 工具路由器，将模型请求的工具调用分发到正确的处理器 | `tools/src/lib.rs` |
| **ToolRegistry** | 工具注册表，维护所有可用工具定义的中央注册处 | `tools/src/lib.rs` |
| **ToolHandler** | 工具处理器，执行具体工具调用逻辑的实现 | `tools/src/lib.rs` |
| **ToolOrchestrator** | 工具编排器，协调多个工具调用的并发执行与结果聚合 | `core/src/codex_thread.rs` |
| **DynamicToolSpec** | 动态工具规格，运行时注册的工具定义（区别于编译时静态工具） | `codex-protocol` |
| **ExecutorFileSystem** | 执行器文件系统 trait，抽象文件操作以支持沙箱环境 | `exec-server/src/lib.rs` |

## App-Server

| 术语 | 定义 | 来源 |
|------|------|------|
| **AppServer** | 应用服务器，提供 JSON-RPC over WebSocket/stdio 接口的后端进程 | `app-server/src/lib.rs` |
| **CodexMessageProcessor** | Codex 消息处理器，处理所有 JSON-RPC 请求的核心分发器 | `app-server/src/codex_message_processor.rs` |
| **ConnectionId** | 连接标识符，标识一个 WebSocket/stdio 客户端连接 | `app-server/src/transport/mod.rs` |
| **ServerNotification** | 服务器通知，AppServer 主动推送给客户端的事件 | `app-server-protocol/src/lib.rs` |

## 状态与持久化

| 术语 | 定义 | 来源 |
|------|------|------|
| **StateRuntime** | 状态运行时，封装 SQLite 连接池和运行时配置的入口类型 | `state/src/runtime.rs` |
| **ThreadMetadata** | 线程元数据，SQLite 中存储的线程摘要信息 | `state/src/lib.rs` |
| **BackfillState** | 回填状态，从 JSONL 文件重建 SQLite 索引的进度追踪 | `state/src/lib.rs` |
| **AgentJob** | Agent 作业，追踪子 Agent 生命周期的作业记录 | `state/src/lib.rs` |
| **LogEntry** | 日志条目，存储在独立 Logs DB 中的结构化日志 | `state/src/lib.rs` |

## 可观测性

| 术语 | 定义 | 来源 |
|------|------|------|
| **OtelProvider** | OpenTelemetry 提供者，统一管理日志、追踪和指标三个 provider | `otel/src/provider.rs` |
| **SessionTelemetry** | 会话遥测，会话级别的结构化事件记录协调器 | `otel/src/events/session_telemetry.rs` |
| **AnalyticsEventsClient** | 分析事件客户端，通过 mpsc channel 异步上报产品分析事件 | `analytics/src/client.rs` |
| **AnalyticsReducer** | 分析聚合器，将原始事件聚合为可上报的批次 | `analytics/src/reducer.rs` |
| **W3cTraceContext** | W3C 追踪上下文，包含 traceparent 和 tracestate 的跨进程追踪标识 | `otel/src/trace_context.rs` |

## 功能标志与配置

| 术语 | 定义 | 来源 |
|------|------|------|
| **Feature** | 功能标志，控制特性开关的枚举变体（60+ 个） | `features/src/lib.rs` |
| **Stage** | 特性阶段，表示功能成熟度：UnderDevelopment → Experimental → Stable → Deprecated → Removed | `features/src/lib.rs` |
| **Features** | 功能集合，经过多源合并后的最终功能标志集 | `features/src/lib.rs` |
| **ConfigToml** | TOML 配置，Codex 配置文件的结构化表示 | `config/src/config_toml.rs` |
| **Config** | 配置，运行时使用的完整配置对象 | `config/src/lib.rs` |
| **ConfigOverrides** | 配置覆盖，命令行参数和环境变量对配置的覆盖 | `config/src/overrides.rs` |
| **EventPersistenceMode** | 事件持久化模式，控制 JSONL 文件中记录的事件粒度（Limited/Extended） | `rollout/src/policy.rs` |

## 补丁与搜索

| 术语 | 定义 | 来源 |
|------|------|------|
| **Hunk** | 补丁块，apply-patch 格式中的单个文件操作（Add/Delete/Update） | `apply-patch/src/parser.rs` |
| **FileSearchSession** | 文件搜索会话，管理基于 nucleo 的模糊文件匹配生命周期 | `file-search/src/lib.rs` |
| **FileMatch** | 文件匹配结果，包含分数、路径、匹配类型和高亮索引 | `file-search/src/lib.rs` |

## 协议与传输

| 术语 | 定义 | 来源 |
|------|------|------|
| **JSON-RPC** | JSON 远程过程调用，AppServer 使用的请求/响应协议 | `app-server-protocol/src/jsonrpc_lite.rs` |
| **SSE (Server-Sent Events)** | 服务器发送事件，模型 API 流式响应的传输格式 | `codex-api/src/sse/mod.rs` |
| **WebSocket** | WebSocket 协议，AppServer 与客户端之间的双向通信通道 | `app-server/src/transport/websocket.rs` |
| **RemoteControl** | 远程控制，允许外部系统（如 ChatGPT）控制 Codex 实例 | `app-server/src/transport/remote_control/mod.rs` |
