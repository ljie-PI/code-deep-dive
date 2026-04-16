# 附录A 关键文件索引

> 所有路径相对于 `codex-rs/`。行数为近似值，基于撰写时的代码快照。

## Core（核心引擎）

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `core/src/lib.rs` | ~200 | Core crate 入口，公共导出 |
| `core/src/codex.rs` | ~800 | Codex 主结构体，会话生命周期管理 |
| `core/src/codex_thread.rs` | ~600 | CodexThread 单线程会话循环 |
| `core/src/codex_delegate.rs` | ~400 | CodexDelegate 回调接口 |
| `core/src/client.rs` | ~500 | ModelClient 实现，API 请求发送 |
| `core/src/client_common.rs` | ~300 | 客户端公共逻辑 |
| `core/src/compact.rs` | ~400 | 上下文压缩（Compaction）逻辑 |
| `core/src/compact_remote.rs` | ~200 | 远程压缩支持 |
| `core/src/context_manager/mod.rs` | ~600 | ContextManager 上下文窗口管理 |
| `core/src/context_manager/history.rs` | ~300 | 历史消息管理 |
| `core/src/context_manager/normalize.rs` | ~200 | 消息归一化 |
| `core/src/context_manager/updates.rs` | ~200 | 上下文更新处理 |
| `core/src/exec.rs` | ~400 | 命令执行核心 |
| `core/src/exec_env.rs` | ~200 | 执行环境配置 |
| `core/src/exec_policy.rs` | ~300 | ExecPolicy 策略执行 |
| `core/src/event_mapping.rs` | ~200 | 事件映射转换 |
| `core/src/function_tool.rs` | ~300 | 函数工具注册 |
| `core/src/hook_runtime.rs` | ~300 | Hook 运行时 |
| `core/src/mcp.rs` | ~400 | MCP 协议集成 |
| `core/src/mcp_tool_call.rs` | ~300 | MCP 工具调用 |
| `core/src/mcp_tool_exposure.rs` | ~200 | MCP 工具暴露策略 |
| `core/src/mcp_tool_approval_templates.rs` | ~100 | MCP 工具审批模板 |
| `core/src/mcp_skill_dependencies.rs` | ~200 | MCP Skill 依赖管理 |
| `core/src/message_history.rs` | ~300 | 消息历史管理 |
| `core/src/network_proxy_loader.rs` | ~200 | 网络代理加载 |
| `core/src/network_policy_decision.rs` | ~150 | 网络策略决策 |
| `core/src/otel_init.rs` | ~100 | OpenTelemetry 初始化 |
| `core/src/landlock.rs` | ~200 | Linux Landlock 沙箱 |

## Core - Agent 子系统

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `core/src/agent/mod.rs` | ~200 | Agent 模块入口 |
| `core/src/agent/agent_resolver.rs` | ~300 | Agent 路由解析 |
| `core/src/agent/control.rs` | ~200 | Agent 控制接口 |
| `core/src/agent/mailbox.rs` | ~150 | Agent 消息邮箱 |
| `core/src/agent/registry.rs` | ~200 | Agent 注册表 |
| `core/src/agent/role.rs` | ~100 | Agent 角色定义 |
| `core/src/agent/status.rs` | ~100 | Agent 状态追踪 |

## Core - Guardian

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `core/src/guardian/mod.rs` | ~100 | Guardian 模块入口 |
| `core/src/guardian/review.rs` | ~400 | 自动审查逻辑 |
| `core/src/guardian/review_session.rs` | ~200 | 审查会话管理 |
| `core/src/guardian/prompt.rs` | ~200 | 审查提示构建 |
| `core/src/guardian/approval_request.rs` | ~150 | 审批请求处理 |

## Core - Memories

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `core/src/memories/mod.rs` | ~100 | 记忆模块入口 |
| `core/src/memories/phase1.rs` | ~300 | 第一阶段记忆提取 |
| `core/src/memories/phase2.rs` | ~300 | 第二阶段记忆合并 |
| `core/src/memories/storage.rs` | ~200 | 记忆存储 |
| `core/src/memories/usage.rs` | ~200 | 记忆使用/召回 |
| `core/src/memories/prompts.rs` | ~150 | 记忆提示模板 |
| `core/src/memories/citations.rs` | ~100 | 记忆引用 |

## Core - Config

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `core/src/config/mod.rs` | ~300 | Config 模块入口 |
| `core/src/config/permissions.rs` | ~200 | 权限配置 |
| `core/src/config/network_proxy_spec.rs` | ~150 | 网络代理规格 |
| `core/src/config/service.rs` | ~200 | 配置服务 |
| `core/src/config_loader/mod.rs` | ~300 | 配置加载器 |

## Core - Plugins

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `core/src/plugins/manager.rs` | ~300 | 插件管理器 |
| `core/src/plugins/manifest.rs` | ~200 | 插件清单解析 |
| `core/src/plugins/marketplace.rs` | ~200 | 插件市场 |
| `core/src/plugins/injection.rs` | ~150 | 插件注入 |
| `core/src/plugins/discoverable.rs` | ~100 | 可发现插件 |

## Config Crate

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `config/src/lib.rs` | ~300 | Config crate 入口 |
| `config/src/config_toml.rs` | ~400 | ConfigToml TOML 配置解析 |
| `config/src/overrides.rs` | ~200 | ConfigOverrides 命令行覆盖 |
| `config/src/merge.rs` | ~200 | 配置合并逻辑 |
| `config/src/mcp_types.rs` | ~200 | MCP 类型定义 |
| `config/src/mcp_edit.rs` | ~200 | MCP 配置编辑 |
| `config/src/permissions_toml.rs` | ~150 | 权限 TOML 配置 |
| `config/src/state.rs` | ~200 | 配置状态管理 |
| `config/src/types.rs` | ~200 | 配置类型定义 |
| `config/src/shell_environment.rs` | ~150 | Shell 环境配置 |
| `config/src/skills_config.rs` | ~100 | Skills 配置 |

## App-Server

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `app-server/src/lib.rs` | ~200 | AppServer crate 入口 |
| `app-server/src/main.rs` | ~100 | 独立服务器入口 |
| `app-server/src/codex_message_processor.rs` | ~800 | CodexMessageProcessor JSON-RPC 消息处理 |
| `app-server/src/message_processor.rs` | ~400 | 通用消息处理器 |
| `app-server/src/thread_state.rs` | ~300 | 线程状态管理 |
| `app-server/src/thread_status.rs` | ~200 | 线程状态查询 |
| `app-server/src/command_exec.rs` | ~300 | 命令执行 API |
| `app-server/src/dynamic_tools.rs` | ~200 | 动态工具管理 |
| `app-server/src/filters.rs` | ~200 | 请求过滤器 |
| `app-server/src/transport/mod.rs` | ~200 | 传输层入口 |
| `app-server/src/transport/websocket.rs` | ~300 | WebSocket 传输 |
| `app-server/src/transport/stdio.rs` | ~200 | stdio 传输 |
| `app-server/src/transport/auth.rs` | ~200 | 传输认证 |
| `app-server/src/transport/remote_control/mod.rs` | ~200 | 远程控制模块 |
| `app-server/src/in_process.rs` | ~200 | 进程内服务器 |

## App-Server Protocol

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `app-server-protocol/src/lib.rs` | ~400 | 协议 crate 入口，所有 RPC 类型定义 |
| `app-server-protocol/src/protocol/v2.rs` | ~600 | V2 协议类型 |
| `app-server-protocol/src/protocol/v1.rs` | ~400 | V1 协议类型（兼容） |
| `app-server-protocol/src/protocol/common.rs` | ~300 | 公共协议类型 |
| `app-server-protocol/src/protocol/mappers.rs` | ~300 | 协议版本映射 |
| `app-server-protocol/src/jsonrpc_lite.rs` | ~200 | JSON-RPC 精简实现 |

## Exec & Sandboxing

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `exec/src/lib.rs` | ~400 | Exec crate 入口 |
| `exec-server/src/lib.rs` | ~300 | ExecServer，ExecutorFileSystem trait |
| `execpolicy/src/lib.rs` | ~400 | ExecPolicy 规则引擎 |
| `linux-sandbox/src/lib.rs` | ~300 | Linux 沙箱（bubblewrap/Landlock） |
| `sandboxing/src/lib.rs` | ~300 | 跨平台沙箱抽象 |
| `windows-sandbox-rs/src/lib.rs` | ~200 | Windows 沙箱入口 |
| `windows-sandbox-rs/src/policy.rs` | ~300 | Windows 沙箱策略 |
| `windows-sandbox-rs/src/token.rs` | ~200 | Windows 受限令牌 |

## Codex API Client

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `codex-api/src/lib.rs` | ~200 | API 客户端入口 |
| `codex-api/src/endpoint/responses.rs` | ~400 | Responses API 端点 |
| `codex-api/src/endpoint/responses_websocket.rs` | ~300 | WebSocket Responses API |
| `codex-api/src/sse/responses.rs` | ~300 | SSE 流解析 |
| `codex-api/src/rate_limits.rs` | ~200 | 速率限制处理 |
| `codex-api/src/provider.rs` | ~200 | Provider 抽象 |

## Tools

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `tools/src/lib.rs` | ~300 | Tools crate 入口 |
| `shell-command/src/lib.rs` | ~300 | Shell 命令工具 |
| `skills/src/lib.rs` | ~200 | Skills 系统入口 |
| `core-skills/src/lib.rs` | ~100 | 核心 Skill 入口 |
| `core-skills/src/manager.rs` | ~300 | Skill 管理器 |
| `core-skills/src/loader.rs` | ~200 | Skill 加载器 |

## State & Persistence

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `state/src/lib.rs` | ~60 | State crate 入口 |
| `state/src/runtime.rs` | 356 | StateRuntime SQLite 运行时 |
| `state/src/migrations.rs` | 29 | 迁移器定义 |
| `thread-store/src/store.rs` | 65 | ThreadStore trait |
| `thread-store/src/types.rs` | 228 | 线程存储类型 |
| `thread-store/src/recorder.rs` | 28 | ThreadRecorder trait |
| `rollout/src/recorder.rs` | 1354 | RolloutRecorder JSONL 录制 |
| `rollout/src/list.rs` | 1291 | 会话发现与列表 |
| `rollout/src/metadata.rs` | 443 | 会话元数据提取 |
| `rollout/src/session_index.rs` | 263 | 线程名称索引 |
| `rollout/src/state_db.rs` | 546 | Rollout 到 SQLite 桥接 |
| `rollout/src/policy.rs` | 224 | 事件持久化策略 |

## Git Utils

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `git-utils/src/ghost_commits.rs` | 1786 | Ghost Commit 创建与恢复 |
| `git-utils/src/info.rs` | 708 | Git 仓库信息查询 |
| `git-utils/src/apply.rs` | 847 | Git apply 操作 |
| `git-utils/src/operations.rs` | 239 | Git 命令封装 |
| `git-utils/src/branch.rs` | 256 | 分支操作 |
| `git-utils/src/lib.rs` | 116 | Git Utils 入口 |

## Analytics & Observability

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `analytics/src/client.rs` | ~500 | AnalyticsEventsClient |
| `analytics/src/events.rs` | ~600 | 事件类型定义 |
| `analytics/src/reducer.rs` | ~200 | AnalyticsReducer 聚合 |
| `analytics/src/facts.rs` | ~300 | AnalyticsFact 输入类型 |
| `otel/src/provider.rs` | 466 | OtelProvider |
| `otel/src/events/session_telemetry.rs` | 1097 | SessionTelemetry |
| `otel/src/trace_context.rs` | 176 | W3C TraceContext |
| `otel/src/otlp.rs` | 272 | OTLP 导出器构建 |

## Models & Features

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `models-manager/src/manager.rs` | 586 | 模型列表管理 |
| `models-manager/src/cache.rs` | 183 | 模型缓存 |
| `features/src/lib.rs` | 993 | Feature 枚举与 Features 解析 |
| `features/src/legacy.rs` | 125 | Legacy 键兼容 |

## Apply-Patch & File-Search

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `apply-patch/src/lib.rs` | 1319 | 补丁应用核心 |
| `apply-patch/src/parser.rs` | 901 | 补丁格式解析 |
| `apply-patch/src/invocation.rs` | 857 | 调用路径处理 |
| `apply-patch/src/seek_sequence.rs` | 163 | 模糊行匹配 |
| `file-search/src/lib.rs` | 1187 | 模糊文件搜索引擎 |

## CLI & TUI

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `cli/src/main.rs` | ~200 | CLI 入口 |
| `cli/src/lib.rs` | ~300 | CLI 库逻辑 |
| `tui/src/tui.rs` | ~500 | TUI 主循环 |
| `tui/src/chatwidget.rs` | ~600 | 聊天组件 |
| `tui/src/lib.rs` | ~200 | TUI 库入口 |
| `tui/src/markdown_render.rs` | ~300 | Markdown 渲染 |

## Connectors & ChatGPT

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `connectors/src/lib.rs` | 549 | 第三方连接器发现与缓存 |
| `chatgpt/src/chatgpt_client.rs` | ~300 | ChatGPT 客户端 |
| `chatgpt/src/lib.rs` | ~100 | ChatGPT crate 入口 |

## MCP

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `codex-mcp/src/lib.rs` | ~100 | MCP crate 入口 |
| `codex-mcp/src/mcp/mod.rs` | ~300 | MCP 协议实现 |
| `codex-mcp/src/mcp_connection_manager.rs` | ~400 | MCP 连接管理器 |
| `codex-mcp/src/mcp_tool_names.rs` | ~100 | MCP 工具名称 |

## Network & Security

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `network-proxy/src/lib.rs` | ~400 | NetworkProxy 入口 |
| `secrets/src/lib.rs` | ~200 | SecretsManager |
| `process-hardening/src/lib.rs` | ~200 | 进程加固 |

## Utilities

| 文件路径 | 约行数 | 说明 |
|----------|--------|------|
| `utils/absolute-path/src/lib.rs` | ~100 | AbsolutePathBuf |
| `utils/output-truncation/src/lib.rs` | ~200 | 输出截断 |
| `utils/stream-parser/src/lib.rs` | ~200 | 流式解析器 |
| `utils/pty/src/lib.rs` | ~200 | PTY 抽象 |
| `utils/home-dir/src/lib.rs` | ~100 | Home 目录检测 |
| `utils/sleep-inhibitor/src/lib.rs` | ~100 | 睡眠抑制器 |
| `async-utils/src/lib.rs` | ~100 | 异步工具 |
