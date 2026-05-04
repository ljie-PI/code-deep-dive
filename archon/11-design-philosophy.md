# 第十一章：设计哲学与扩展机制

> Archon 的核心设计决策和如何扩展系统。

## 11.1 核心设计原则

### KISS — Keep It Simple

- 单一可执行文件（Bun 编译二进制）
- SQLite 零配置默认
- `archon workflow run <name> <msg>` 一行启动
- 隔离即默认——不需要理解 worktree 也能安全使用

### YAGNI — 不预设未来需求

- 单用户工具，无多租户复杂性
- 没有权限系统——用环境变量白名单控制
- 没有队列系统——ConversationLockManager 足够
- 没有插件系统——YAML 工作流本身就是扩展机制

### 平台无关

- `IPlatformAdapter` 接口统一所有平台
- 同一个 `handleMessage()` 处理所有来源
- 会话、Session、隔离环境跨平台共享

### Git 作为一等公民

- 让 Git 做 Git 擅长的事（冲突检测、分支管理）
- 暴露 Git 错误给用户（不吞掉 "has uncommitted changes"）
- 信任 Git 的天然护栏（拒绝删除有未提交变更的 worktree）
- 绝不执行 `git clean -fd`

### Fail Fast

- 不可支持的路径 → 抛错误，不静默降级
- 不安静地扩展权限或能力
- AI 运行时的静默回退可能导致不安全或昂贵的行为

## 11.2 关键设计决策

### 为什么 Bun 而不是 Node.js？

- `bun build --compile` 生成单二进制文件
- 内置 SQLite（`bun:sqlite`）
- 内置 `.env` 自动加载
- 工作区（workspaces）原生支持
- 比 Node.js 更快的启动和执行

### 为什么 Hono 而不是 Express？

- `@hono/zod-openapi` 集成——schema 验证和 API 文档一次搞定
- 零配置 CORS
- 内置 SSE 支持（`hono/streaming`）
- 轻量级，与 Bun 配合良好

### 为什么 YAML DAG 而不是代码？

- 非开发者也能阅读和编辑
- 声明式——描述"做什么"而非"怎么做"
- 可验证——Zod schema 在加载时捕获错误
- 可组合——节点通过 `$nodeId.output` 传递数据

### 为什么 Worktree 而不是 Docker？

- Worktree 创建廉价（毫秒级 vs 容器的秒级）
- 共享 `.git` 目录，节省磁盘空间
- 本地开发者已有 Git，无需 Docker
- 确定性端口分配避免冲突

### 为什么依赖注入？

`@archon/workflows` 通过 `WorkflowDeps` 接口接收外部依赖，而非直接 import `@archon/core`：

- 消除循环依赖
- 引擎可独立测试（mock deps）
- 未来可替换数据库或 AI 客户端
- 保持包的职责清晰

## 11.3 扩展机制

### 添加新平台适配器

1. 实现 `IPlatformAdapter` 接口
2. 在 `packages/server/src/index.ts` 中条件初始化
3. 注册 `onMessage` 回调到 `handleMessage()`

### 添加新 AI Provider

1. 在 `packages/providers/src/<provider-id>/` 下新建 provider 实现：
   - `provider.ts`：实现 `IAgentProvider`（`sendQuery`、`getType`、`getCapabilities`）
   - `capabilities.ts`：声明支持的模型与功能（`output_format`、`hooks`、`mcp`、`skills` 等）
   - 可选：`binary-resolver.ts`、`config.ts`
2. 在 `packages/providers/src/registry.ts` 的 `registerBuiltinProviders()` 中注册（或社区 provider 通过自己的 `registration.ts` 暴露注册函数，由调用方按需调用）
3. `sendQuery()` 必须返回 `AsyncGenerator<MessageChunk>` 流

### 添加新工作流节点类型

1. 在 `packages/workflows/src/schemas/dag-node.ts` 中添加 schema
2. 在 `dag-executor.ts` 中添加执行逻辑
3. 更新 `loader.ts` 中的互斥检查

### 添加新斜杠命令

1. 在 `packages/core/src/handlers/command-handler.ts` 中添加 case
2. 返回 `CommandResult`
3. 如果命令应触发工作流，设置 `result.workflow`

### 添加自定义工作流

在仓库的 `.archon/workflows/` 中创建 YAML 文件——引擎自动发现。

### 添加自定义命令

在仓库的 `.archon/commands/` 中创建 `.md` 文件——作为 AI 执行的提示模板。

## 11.4 安全设计

### Env Leak Gate ⚠️ 当前状态

**设计目标**：防止 AI subprocess 继承目标仓库的敏感环境变量。

**当前实现状态（v0.3.x）**：
- 数据库层面的 `allow_env_keys` 同意位仍保留，PATCH `/api/codebases/:id` 仍可设置；
- 但运行时的"启动扫描 + 主动拦截"已在 provider 抽离时被移除（参见 `packages/providers/src/claude/provider.ts:921` 的 `TODO(#1135)`）；
- 旧版 `packages/core/src/utils/env-leak-scanner.ts` 已删除。

也就是说，`allow_env_keys = false` 当前并不会阻止 Claude/Codex subprocess 继承父进程的 env，仅作为 UI 同意位记录。**重新启用主动拦截是 issue #1135 的待办项**。在此之前，请确保通过 CLI/服务器进程本身的环境（而非目标仓库的 `.env`）来传递敏感变量，并依靠 `stripCwdEnv()`（仍然有效）防止 CWD `.env` 自动加载。

### Webhook 签名验证

所有 webhook 端点使用平台特定的签名验证（HMAC SHA-256），并使用 `timingSafeEqual` 防止时序攻击。

### 凭证清洗

- `sanitizeCredentials()` 在日志输出前清洗 API key
- Token 仅显示前 8 个字符
- 永不记录用户消息内容或 PII

### CWD 环境清洗

`stripCwdEnv()` 在任何模块读取 `process.env` 之前运行，防止：
- Bun 自动加载的 CWD `.env` 泄漏
- 嵌套 Claude Code 会话标记导致的死锁

## 11.5 本章关键文件

| 文件 | 职责 |
|------|------|
| `packages/core/src/types/index.ts` | IPlatformAdapter 接口 |
| `packages/providers/src/types.ts` | IAgentProvider / ProviderCapabilities 接口 |
| `packages/providers/src/registry.ts` | Provider 注册表（添加新 AI Provider 的入口） |
| `packages/workflows/src/deps.ts` | WorkflowDeps 依赖注入类型 |
| `packages/workflows/src/schemas/dag-node.ts` | DAG 节点类型系统 |
| `packages/paths/src/strip-cwd-env.ts` | 环境变量安全清洗 |
| `packages/providers/src/claude/provider.ts` | ClaudeProvider（含 `TODO(#1135)` env-leak 复活点） |
