# 4. 权限与安全

三者的安全哲学截然不同：Claude Code 是**纵深防御** (规则→AI分类器→沙箱)，OpenCode 是**规则匹配+用户确认**，Codex 是**沙箱优先** (OS级容器隔离)。

## 4.1 架构对比

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **裁决类型** | Allow / Ask / Deny / Passthrough (4种) | allow / deny / ask (3种) | Allow / Prompt / Forbidden (3种) |
| **规则来源** | 8 种 (session, cliArg, command, 5种 Settings) | 3 层 (Agent 配置, 会话覆盖, 运行时批准) | 6 层 (MDM, System, User, Project, SessionFlags, Legacy) |
| **匹配算法** | 多步流水线 (工具特定→规则→AI分类→模式) | `findLast` (last-match-wins, 15 行) | 前缀规则匹配 + 启发式函数 |
| **AI 分类器** | 双阶段 (快速 + 思考) | 无 | 无 |
| **权限模式** | 7 种 (default/plan/acceptEdits/bypass/dontAsk/auto/bubble) | 无模式 (Agent permission ruleset) | 5 种 (UnlessTrusted/OnFailure/OnRequest/Granular/Never) |
| **沙箱** | `@anthropic-ai/sandbox-runtime` 包 | 无 OS 级沙箱 | 原生: Seatbelt(macOS) + Landlock+Bwrap(Linux) + RestrictedToken(Windows) |
| **网络沙箱** | 通过 sandbox-runtime 包 | 无 | Proxy + 策略规则 |
| **评估文件** | `permissions.ts` (多文件) | `evaluate.ts` (15 行) | `execpolicy/` crate + `guardian/` 模块 |

## 4.2 Claude Code: 工业级安全框架

### 权限评估流水线

```
canUseTool()
  ├── 1. 检查 deny 规则
  ├── 2. tool.checkPermissions() (工具特定检查)
  ├── 3. allow 规则匹配
  ├── 4. ask 规则匹配
  ├── 5. 模式决策 (7 种模式)
  ├── 6. AI 分类器 (auto 模式, 双阶段)
  └── 7. 用户提示 (fallback)
```

### Bash 安全分析
- `bashClassifier.ts` — 模式匹配 allow/deny
- `dangerousPatterns.ts` — 硬编码危险命令模式
- `shellRuleMatching.ts` — 解析 Shell 命令匹配权限规则
- `yoloClassifier.ts` — AI 分类器，审查对话上下文 + 待执行操作，调用 Claude API 决定 block/allow
  - 双阶段: 快速判断 + 不确定时启用思考模式

### AI 安全分类器
Claude Code 独有的 `yoloClassifier.ts`:
- 输入: 完整对话记录 + 待执行的工具调用
- 输出: block / allow
- 实现: 调用 Claude API，先快速判断，不确定时启用思考模式 (thinking)
- 仅在 `auto` 权限模式下启用

### 沙箱
- 使用 `@anthropic-ai/sandbox-runtime` npm 包
- 可配置: `FsReadRestrictionConfig`, `FsWriteRestrictionConfig`, `NetworkRestrictionConfig`
- 与 settings 系统和 per-tool 配置集成

## 4.3 OpenCode: 轻量级规则引擎

### 核心评估 (仅 15 行)

```typescript
// permission/evaluate.ts
export function evaluate(permission: string, pattern: string, ...rulesets: Rule[][]): Rule {
  const rules = rulesets.flat()
  const match = rules.findLast(
    (rule) => Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern),
  )
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

### 权限流程
```
Permission.ask()
  ├── evaluate() (规则匹配, last-match-wins)
  ├── allowed/denied → 立即返回
  └── ask → 创建 Deferred, 发布事件到 Bus, 等待 UI 回复
       └── 回复选项: once / always / reject
```

- `always` 选项自动将规则加入会话级批准列表，并自动解决其他相同类型的待审批请求
- **无 AI 分类器，无 OS 沙箱** — 安全完全依赖规则匹配 + 用户判断

## 4.4 Codex: OS 级沙箱隔离

### Guardian 审批系统

```
GuardianApprovalRequest
  ├── Shell — 命令 + 沙箱权限 + 额外权限 + 理由
  ├── ExecCommand — 可执行文件 + 参数
  ├── Execve — Unix 直接执行
  ├── ApplyPatch — 文件补丁
  ├── NetworkAccess — 网络请求
  └── McpToolCall — MCP 工具调用 (含 destructive/open_world/read_only 提示)
```

### 执行策略 (`execpolicy/` crate)
- `Policy` 结构体: 程序名 → 前缀规则映射
- `PrefixRule`: 匹配命令前缀 (如 `git status` → Allow, `rm -rf /` → Forbidden)
- `Evaluation::from_matches()`: 聚合匹配结果，取最严格决策
- 支持网络规则: host/protocol 匹配

### 沙箱实现 (`sandboxing/` crate)
```
SandboxPolicy
  ├── DangerFullAccess — 无限制
  ├── ReadOnly — 只读文件系统
  ├── ExternalSandbox — 外部沙箱
  └── WorkspaceWrite — 工作区写入

平台实现:
  ├── macOS: Seatbelt (seatbelt.rs + .sbpl 策略文件)
  ├── Linux: Landlock + Bubblewrap (landlock.rs, bwrap.rs) + 专用 linux-sandbox crate
  └── Windows: Restricted Token (SandboxType::WindowsRestrictedToken)
```

- `SandboxManager`: 根据策略构建沙箱化命令，设置环境变量、工作目录
- 网络沙箱: 通过代理 (`NetworkProxy`) 实现
- **工具执行可重试 + 沙箱升级**: 如果在严格沙箱中失败，可尝试升级沙箱权限重试

### 审批模式

```rust
pub enum AskForApproval {
    UnlessTrusted,    // 仅自动批准已知安全的只读命令
    OnFailure,        // (已废弃) 自动批准 + 沙箱 + 失败时升级
    OnRequest,        // 模型决定何时请求批准 (默认)
    Granular(GranularApprovalConfig),  // 细粒度: sandbox/rules/skill/request/mcp 分别控制
    Never,            // 从不询问，失败直接返回模型
}
```

## 4.5 评价

| 维度 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **安全深度** | 最深 (规则+AI+沙箱) | 最浅 (仅规则+用户确认) | 最深 (规则+OS沙箱) |
| **AI 辅助** | 独有: 双阶段 AI 分类器 | 无 | 无 |
| **OS 隔离** | npm 包封装 | 无 | 原生实现, 三平台 |
| **企业管控** | managed settings | managed config + MDM | managed config + MDM |
| **灵活性** | 7 种模式, 高度可配 | 简单直接 | 5 种模式 + 细粒度配置 |
| **实现复杂度** | 高 (多文件, 多步流水线) | 低 (15 行核心) | 高 (多 crate, 多平台) |

**总结**: Claude Code 用 AI 分类器弥补规则匹配的盲区，Codex 用 OS 级沙箱从根本上限制进程能力，OpenCode 信任用户判断。三种哲学各有优势——Claude Code 最"智能"，Codex 最"硬核"，OpenCode 最"轻量"。
