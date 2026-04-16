# 6. 上下文管理与压缩

## 6.1 上下文窗口

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **默认窗口** | 200,000 tokens | 取决于模型配置 | 取决于模型配置 |
| **大窗口** | 1,000,000 tokens (`[1m]` 后缀) | 取决于模型配置 | 取决于模型配置 |
| **有效窗口** | 总窗口 - min(输出上限, 20K) | 总窗口 - maxOutput - 预留缓冲 | 总窗口 × 95% |
| **Token 计数** | API usage (精确) + 启发式估算 (fallback) | 4字符/token 启发式 + API usage | 4字节/token 启发式 + API usage |
| **精确计数** | 支持: Anthropic API `countTokens`, Bedrock, Vertex | 不支持 | 不支持 |
| **环境变量覆盖** | `CLAUDE_CODE_CONTEXT_WINDOW_OVERRIDE` | 无 | 无 |

## 6.2 压缩策略对比

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **压缩触发** | 有效窗口 - 13K (`AUTOCOMPACT_BUFFER_TOKENS`) | tokenCount >= usableWindow | total_usage >= auto_compact_limit (per model) |
| **压缩方式** | LLM 摘要 + prompt cache 共享 + 部分压缩 | 两层: prune (修剪旧工具输出) → summarize (LLM 摘要) | LLM 摘要 + 用户消息保留 |
| **摘要模型** | 使用 forked agent (复用 prompt cache) | compaction agent (可配置模型) | 同主模型 |
| **压缩自身溢出** | PTL 重试: 逐步删除最老的 API 轮次 (最多 3 次) | 查找最后用户消息, 剥离媒体, 重放 | 逐个删除最老历史项 |
| **压缩后恢复** | 重读最近 5 个文件 (50K上限), 重注入 plans/skills/tools/MCP | 无显式恢复 | 重注入初始上下文 (系统指令) |
| **媒体处理** | 压缩前剥离图片和文档 | 压缩溢出时剥离 | 不明确 |
| **断路器** | 连续 3 次失败后停止 | 无 | 逐项删除后重试 |
| **远程压缩** | 无 | 无 | 支持 (Azure 等服务端压缩) |

## 6.3 Claude Code 的压缩系统

### 阈值体系
```
┌─────────────────────────────────────────────┐
│              总窗口 (200K / 1M)              │
├─────────────────────────────────────────────┤
│         有效窗口 = 总窗口 - 8K~20K          │
├─────────────────────────────────────────────┤
│     自动压缩阈值 = 有效窗口 - 13K           │
├─────────────────────────────────────────────┤
│      阻塞限制 = 有效窗口 - 3K               │
└─────────────────────────────────────────────┘
```

### 压缩流程 (`compact.ts`)
1. 剥离图片和文档
2. 使用 forked agent 生成摘要 (复用主对话 prompt cache)
3. 如果压缩自身 PTL，逐步删除最老 API 轮次，重试 (最多 3 次)
4. 压缩后恢复:
   - 重读最近 5 个访问的文件 (每文件最多 5K tokens, 总共最多 50K tokens)
   - 重注入 plans, skills, deferred tools, agent listings, MCP instructions
5. 可选: session memory compaction (作为全量压缩的替代)
6. 可选: partial compaction (只压缩 pivot 消息之前/之后的部分)

### Prompt Cache 优化
- Fork 子 Agent 产生字节一致的 API 请求前缀 → cache hit
- `promptCacheBreakDetection.ts` 追踪系统提示、工具、betas、模型的哈希值
- 压缩后重置 cache baseline

## 6.4 OpenCode 的两层压缩

### 常量
| 常量 | 值 | 含义 |
|------|-----|------|
| `PRUNE_PROTECT` | 40,000 tokens | 最近工具输出保留量 |
| `PRUNE_MINIMUM` | 20,000 tokens | 触发修剪的最小节省 |
| `COMPACTION_BUFFER` | 20,000 tokens | 上下文限制预留 |
| Doom loop 阈值 | 3 | 连续相同工具调用中断 |

### 压缩流程 (`compaction.ts`)
1. **Pruning** (第一层): 从后向前遍历消息 parts，保留 40K tokens 的最近工具输出，清除更老的工具输出 (标记为 `compacted`)。仅在节省 ≥ 20K tokens 时执行。`skill` 工具受保护不被修剪
2. **Summarization** (第二层): 使用 compaction agent 生成结构化摘要 (Goal, Instructions, Discoveries, Accomplished, Relevant files)
3. **插件扩展**: 插件可通过 `experimental.session.compacting` hook 注入上下文或替换压缩提示

## 6.5 Codex 的压缩系统

### 压缩流程 (`compact.rs`)
1. 发送对话到模型，使用 `SUMMARIZATION_PROMPT` 生成摘要
2. 构建压缩历史: 保留截断的用户消息 (最近优先, 预算 `COMPACT_USER_MESSAGE_MAX_TOKENS = 20,000`)
3. 附加摘要 (带 `SUMMARY_PREFIX`)
4. 如果压缩自身溢出: 逐个删除最老历史项后重试
5. 重注入初始上下文 (系统指令) — 放在最后一条用户消息或摘要之前
6. Ghost snapshots 在压缩后保留

### 特殊功能
- **远程压缩**: 对于某些 Provider (如 Azure)，委托给服务端执行 (`run_inline_remote_auto_compact_task()`)
- **压缩分析**: `CompactionAnalyticsAttempt` 结构体追踪触发原因、实现方式、阶段、token 前后变化、耗时
- **用户警告**: "Long threads and multiple compactions can cause the model to be less accurate"

## 6.6 评价

| 维度 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **Token 计数精度** | 最高 (API 精确计数 + 多 Provider 支持) | 中 (4字符启发式) | 中 (4字节启发式) |
| **压缩层次** | 最丰富 (摘要 + prompt cache + 部分压缩 + session memory) | 两层 (prune + summarize) | 单层 (摘要 + 用户消息保留) |
| **压缩后恢复** | 最完善 (文件/plans/skills/tools/MCP 全部重注入) | 无 | 初始上下文重注入 |
| **Prompt Cache** | 深度优化 (fork sharing + break detection) | 无显式策略 | 无显式策略 |
| **远程压缩** | 无 | 无 | 支持 (Azure) |
| **可扩展性** | 低 (内部实现) | 高 (插件 hook) | 低 (内部实现) |
