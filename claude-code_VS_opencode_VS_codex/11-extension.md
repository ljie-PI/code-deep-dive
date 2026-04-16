# 11. 扩展机制

## 11.1 对比

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **Hook 系统** | 配置式 (settings.json, Shell 命令/prompt/agent) | 代码式 (npm 插件 → Hooks 接口) | Skill 注入系统 |
| **Hook 事件** | PreToolUse, PostToolUse, Notification, Stop 等 | tool.execute.before/after, command.execute.before, chat.messages.transform 等 | Pre/post tool hooks |
| **插件系统** | 无公开 API (内部 `builtinPlugins.ts`) | 完整 npm 插件系统 | Plugin crate (Rust) |
| **插件能力** | — | 工具, Agent, 命令, Provider, 事件钩子, 认证 | Skills, MCP 服务器, App 连接器 |
| **事件总线** | 无 (React hooks + 模块单例) | Effect PubSub (类型化, Zod schema) | tokio channels (mpsc) |
| **SDK** | 无公开 SDK | @opencode-ai/sdk (npm) | Protocol crate (Rust) + TS 类型生成 |
| **MCP 服务器** | Claude Code 可作为 MCP 服务器 | 仅客户端 | 专用 MCP 服务器二进制 |
| **Slash 命令** | 109 个内置 + 自定义 Skills | Skills + 内置命令 | Skills 系统 |

## 11.2 Claude Code: 声明式 Hooks

### Hook 配置 (`settings.json`)
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "type": "command",
        "command": "bash check-tool.sh",
        "if": "Bash(git *)",
        "timeout": 5000
      }
    ]
  }
}
```

### Hook 类型
- `command` — Shell 命令 (返回 JSON 控制行为)
- `prompt` — LLM 评估
- `agent` — Agent 执行

### 特点
- 声明式配置，非代码
- 支持 `if` 条件 (使用权限规则语法)
- 支持 `once` (一次性) 模式
- 支持超时和异步执行
- **无公开插件 API** — 内部有 `builtinPlugins.ts`，但不面向用户

### Skills 系统
- Markdown + YAML frontmatter (`SKILL.md`)
- 搜索: `.claude/skills/`, Claude Code 内部 skills
- 支持 `$ARGUMENTS`, `$1`, `$2` 模板变量
- 支持 `@file` 引用
- Frontmatter 可限制 `allowed_tools`

## 11.3 OpenCode: npm 插件生态

### 插件接口
```typescript
type Plugin = (input: PluginInput) => Promise<Hooks>
```

### Hooks 接口 (极其丰富)
```
event              — 事件监听
config             — 配置修改
tool               — 自定义工具定义
auth               — 认证钩子
provider           — 自定义 Provider
chat.message       — 消息转换
chat.params        — 参数修改
chat.headers       — 请求头修改
permission.ask     — 权限拦截
command.execute.before  — 命令前置
tool.execute.before     — 工具执行前
tool.execute.after      — 工具执行后
shell.env          — Shell 环境变量
```

### 内置插件
- Codex auth, GitHub Copilot auth, GitLab auth, Poe auth, Cloudflare auth

### 特点
- npm 包或本地路径加载
- `PluginLoader` 可运行时安装和加载
- 代码级扩展 — 可注入工具、Provider、Agent
- Effect PubSub 事件总线 (类型化, Zod schema, 支持通配符)
- `@opencode-ai/sdk` 公开 SDK

## 11.4 Codex: 三层扩展

### Plugin 系统 (`plugin/` crate)
- `PluginId`, `PluginLoadOutcome`, `PluginCapabilitySummary`
- 插件提供: Skills, MCP 服务器, App 连接器

### Skills 系统 (`core-skills/`)
- 配置规则 + 环境变量依赖
- 注入系统 (从磁盘和远程加载)
- Mention 语法

### Connectors (`connectors/`)
- App 连接器系统
- 动态目录列表发现可用应用
- 桥接外部集成

### 特点
- 三层: Plugins → Skills → Connectors
- Pre/post tool hooks
- 无通用事件总线 (tokio channels 用于内部通信)

## 11.5 评价

| 维度 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **扩展深度** | 浅 (声明式 hooks) | 最深 (代码级插件) | 中 (plugin+skills+connectors) |
| **易用性** | 高 (JSON 配置即可) | 中 (需写 npm 包) | 中 (需写 Rust) |
| **事件系统** | 无 | 最丰富 (PubSub) | 内部 channels |
| **SDK** | 无 | 有 (npm) | 有 (Rust + TS) |
| **社区友好度** | 低 (无插件 API) | 最高 (npm 生态) | 中 (Rust 门槛) |

**总结**: OpenCode 的扩展性**远超**另两者 — 代码级插件可修改几乎一切，npm 生态降低门槛。Claude Code 通过 MCP 服务器角色和声明式 Hooks 提供有限但简单的扩展。Codex 的三层架构 (plugin/skills/connectors) 在 Rust 生态内提供了完整但门槛较高的扩展能力。
