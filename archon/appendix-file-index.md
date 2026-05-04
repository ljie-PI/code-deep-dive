# 附录 A：关键文件索引

> 行数为 v0.3.10（HEAD `88d01099`）快照。

## 按重要性排序

### Tier 1 — 核心引擎（必读）

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/workflows/src/dag-executor.ts` | 3,184 | DAG 工作流执行引擎 |
| `packages/core/src/orchestrator/orchestrator-agent.ts` | 1,557 | AI 会话编排器（handleMessage 入口） |
| `packages/core/src/handlers/command-handler.ts` | 1,150 | 斜杠命令处理 |
| `packages/isolation/src/providers/worktree.ts` | 1,227 | Worktree 隔离提供者 |
| `packages/providers/src/claude/provider.ts` | 1,055 | ClaudeProvider（v0.3.x 抽离，含 `TODO(#1135)`） |
| `packages/providers/src/codex/provider.ts` | 665 | CodexProvider（v0.3.x 抽离） |
| `packages/workflows/src/schemas/dag-node.ts` | 638 | DAG 节点 Zod schema（7 种节点类型定义） |
| `packages/workflows/src/executor.ts` | 831 | 工作流执行协调器 |
| `packages/workflows/src/router.ts` | 266 | 意图路由和工作流匹配 |
| `packages/workflows/src/loader.ts` | 504 | YAML 解析和验证 |

### Tier 2 — 平台与接口

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/server/src/routes/api.ts` | 2,698 | REST API 路由定义 |
| `packages/cli/src/commands/setup.ts` | 1,928 | CLI 交互式设置向导 |
| `packages/cli/src/commands/workflow.ts` | 1,129 | CLI workflow 子命令 |
| `packages/adapters/src/forge/github/adapter.ts` | 952 | GitHub 适配器 |
| `packages/core/src/db/workflows.ts` | 1,007 | 工作流数据库操作 |
| `packages/adapters/src/community/forge/gitea/adapter.ts` | 912 | Gitea 适配器（社区） |
| `packages/adapters/src/community/forge/gitlab/adapter.ts` | 798 | GitLab 适配器（社区） |
| `packages/server/src/index.ts` | 729 | 服务器启动和适配器初始化 |
| `packages/cli/src/cli.ts` | 659 | CLI 入口和命令分发 |

### Tier 3 — 基础设施

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/paths/src/archon-paths.ts` | 494 | 目录路径计算 |
| `packages/paths/src/telemetry.ts` | 246 | 遥测/诊断（v0.3.x 新增） |
| `packages/paths/src/env-loader.ts` | 83 | `~/.archon/.env` 加载（v0.3.x 新增） |
| `packages/core/src/types/index.ts` | 194 | 核心类型（IPlatformAdapter） |
| `packages/providers/src/types.ts` | 316 | IAgentProvider / ProviderCapabilities |
| `packages/providers/src/registry.ts` | 161 | Provider 注册表 |
| `packages/isolation/src/resolver.ts` | 561 | 6 步隔离解析引擎 |
| `packages/workflows/src/executor-shared.ts` | 532 | 变量替换和错误分类 |
| `packages/web/src/lib/api.generated.d.ts` | 2,584 | 生成的 API 类型 |
| `migrations/000_combined.sql` | 314 | 完整数据库 schema |

### Tier 4 — 支撑模块

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/workflows/src/workflow-discovery.ts` | 372 | 工作流文件系统发现 |
| `packages/workflows/src/script-discovery.ts` | 170 | `script:` 节点脚本发现（v0.3.x 新增） |
| `packages/workflows/src/event-emitter.ts` | 261 | 可观测性事件 |
| `packages/workflows/src/condition-evaluator.ts` | 174 | when 条件求值 |
| `packages/paths/src/logger.ts` | 119 | Pino 结构化日志 |
| `packages/paths/src/strip-cwd-env.ts` | 110 | CWD 环境变量清洗 |
| `packages/paths/src/update-check.ts` | 152 | 版本更新检查 |
| `packages/isolation/src/errors.ts` | 141 | 隔离错误分类 |
| `packages/core/src/state/session-transitions.ts` | 89 | Session 状态机 |
