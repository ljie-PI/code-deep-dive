# 9. MCP 集成

## 9.1 对比

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **角色** | 客户端 + 服务器 (双角色) | 仅客户端 | 客户端 + 服务器 (双角色) |
| **SDK** | 官方 TS MCP SDK | 官方 TS MCP SDK | 自研 Rust 实现 |
| **传输** | stdio, SSE, HTTP, WebSocket, SDK, IDE 变体 (7种) | stdio (local), HTTP+SSE (remote) (2种) | stdio, StreamableHTTP (2种) |
| **OAuth** | 支持 (含 XAA 跨应用访问) | 支持 (clientId/Secret/scope/redirectUri) | 支持 (scope 发现 + 无 scope 重试) |
| **工具命名** | `mcp__<server>__<tool>` | `<server>:<tool>` (冒号分隔) | `mcp__<server>__<tool>` |
| **工具排序** | 分区排序 (MCP 在内置之后, 保护 prompt cache) | 动态添加到工具列表 | 合并到 ToolRouter |
| **延迟加载** | 支持 (`shouldDefer` + `ToolSearchTool`) | 不支持 | 支持 (`defer_loading`) |
| **资源** | ListMcpResourcesTool + ReadMcpResourceTool | MCP.resources() (列表+读取) | list_mcp_resources + read_mcp_resource |
| **配置位置** | `.mcp.json` (local), settings (user/project), managed-mcp.json (enterprise) | config `mcp` 字段 | `config.toml` `[mcp_servers]` 表 |
| **配置范围** | 7 种 (local/user/project/dynamic/enterprise/claudeai/managed) | 单一 `mcp` 配置键 (跨层合并) | TOML 表 (跨层合并) |
| **超时** | 可配置 | 默认 30 秒 | 可配置 |

## 9.2 Claude Code: MCP 双角色

### 作为客户端
- 使用 `@modelcontextprotocol/sdk/client`
- 7 种传输: `stdio`, `sse`, `sse-ide`, `http`, `ws`, `sdk`, `ws-ide`, `claudeai-proxy`
- 工具命名: `mcp__<server>__<tool>`
- `assembleToolPool()` 中 MCP 工具在内置工具**之后**排列，保护 prompt cache
- 支持 `shouldDefer` 延迟加载 — 通过 `ToolSearchTool` 按需发现
- 工具去重: 内置工具优先于同名 MCP 工具

### 作为服务器
- `createComputerUseMcpServer` — 将计算机操作工具暴露为 MCP 服务器
- 使用 `StdioServerTransport`
- 允许 IDE 等客户端调用 Claude Code 的工具

### 配置范围 (7 种)
1. `local` — `.mcp.json` (项目级)
2. `user` — 全局配置
3. `project` — 项目配置
4. `dynamic` — 运行时添加
5. `enterprise` — `managed-mcp.json`
6. `claudeai` — Claude.ai 代理
7. `managed` — settings-based

## 9.3 OpenCode: 纯客户端

- 使用 `@modelcontextprotocol/sdk/client`
- 2 种传输: `local` (stdio) 和 `remote` (StreamableHTTP + SSE fallback)
- 工具命名: `<sanitized_server>:<sanitized_tool>` (冒号分隔, 与另两者不同)
- 支持 `ToolListChangedNotification` 动态更新
- REST API 路由 `/mcp` 提供状态查询和管理 (但不是标准 MCP 服务器)
- 完整 OAuth: `clientId`, `clientSecret`, `scope`, `redirectUri`

## 9.4 Codex: Rust 原生 MCP

### 作为客户端
- 自研 Rust 实现 (`codex-mcp` crate)
- `McpConnectionManager` 管理连接生命周期
- 工具名清洗: 确保符合 `[a-zA-Z0-9_-]+` (Responses API 兼容性)
- 支持工具列表变更通知

### 作为服务器
- 专用 MCP 服务器二进制 (`mcp-server/`)
- 暴露文件编辑、命令执行等工具给外部 MCP 客户端
- 允许其他工具链接入 Codex 的能力

### 配置
- TOML `[mcp_servers.<name>]`
- 支持 `env_vars` 透传环境变量
- `McpServerIdentity` 跨配置层去重
- `enabled/disabled` 状态追踪 (含禁用原因)

## 9.5 评价

| 维度 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **传输支持** | 最丰富 (7种) | 最精简 (2种) | 精简 (2种) |
| **服务器能力** | 有 (计算机操作) | 无 | 有 (专用二进制) |
| **配置灵活性** | 最高 (7种范围) | 中等 | 中等 |
| **Prompt Cache** | 分区排序保护 | 无优化 | 无优化 |
| **延迟加载** | 支持 | 不支持 | 支持 |
