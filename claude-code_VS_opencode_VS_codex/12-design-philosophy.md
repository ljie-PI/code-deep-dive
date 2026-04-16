# 12. 设计哲学

## 12.1 核心理念对比

| 理念 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **架构风格** | 单体进程，功能集中 | 微服务化 monorepo, Server-Client 分离 | 模块化 Rust workspace (~50 crates) |
| **安全哲学** | 纵深防御 (规则→AI分类器→沙箱) | 规则匹配 + 用户确认 | 沙箱优先 (OS级容器隔离) |
| **扩展策略** | Hooks (声明式) + MCP 双角色 | 插件系统 (代码级 npm 包) | Plugins + Skills + Connectors |
| **Provider 策略** | 少而精 (6种, 深度适配) | 多而广 (15+, AI SDK 统一抽象) | OpenAI 为主 + 可配置 |
| **状态管理** | 全局可变状态 + 函数组合 | Effect 不可变 Service/Layer | Arc + Mutex (Rust 所有权) |
| **数据持久化** | 文件系统 (JSONL) | 数据库 (SQLite) | 双存储 (JSONL + SQLite) |
| **UI-Engine 耦合** | 紧密 (同进程, REPL.tsx 6K行) | 松散 (Worker 线程 + HTTP/WS) | 松散 (异步任务 + 独立 crate) |
| **多端支持** | CLI 为主, IDE 扩展 | CLI + Web + Desktop (Tauri/Electron) | CLI + App Server (WebSocket) |
| **Build vs Buy** | Build everything (自研 API 层/Ink fork/vim) | Buy where possible (AI SDK/Hono/Drizzle/Effect) | Build core, buy infra (自研沙箱/协议, 用 tokio/ratatui/axum) |

## 12.2 Server & API 层

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **HTTP 服务器** | 无 (stub 实现) | Hono (完整 REST+WS) | Axum (WebSocket) |
| **API 协议** | 无 (stdio) | REST + WebSocket + OpenAPI | JSON-RPC over WebSocket |
| **SDK** | 无 | @opencode-ai/sdk | Rust client + TS 类型 |
| **多客户端** | 不支持 | 支持 | 支持 |
| **Headless** | `--print` 参数 | `serve` 模式 | App Server 协议 |
| **Desktop** | 无 | Tauri + Electron | 无 |
| **Web UI** | 无 | Astro Web 应用 | 无 |
| **IDE** | VS Code (MCP bridge) | Zed 扩展 | 协议驱动 |

## 12.3 各自优势

### Claude Code 的优势
- **更成熟的安全体系**: AI 分类器 + OS 沙箱，三者中唯一有 AI 辅助安全判断
- **更精细的上下文管理**: 混合 Token 计数 (API精确+启发式), 多层压缩, prompt cache 深度优化
- **更强大的多 Agent 架构**: 协调器模式 + Swarm + fork (字节一致 cache 共享)
- **更多内置工具**: 40+ vs 17 / 25+
- **Prompt Cache 优化**: 分区排序、fork 共享、break detection — 三者中最深

### OpenCode 的优势
- **Provider 支持最广**: 15+ 通过 AI SDK 统一抽象
- **架构最现代**: Effect DI + Server-Client + monorepo
- **插件生态最强**: 代码级 npm 插件，可注入工具/Provider/Agent
- **多端支持**: Web + Desktop + TUI + SDK
- **数据查询最灵活**: SQLite + 会话 fork
- **开源，可 Fork 可定制**
- **主题最丰富**: 33 种命名主题

### Codex 的优势
- **性能最高**: Rust 原生，多线程 tokio，零 GC 开销
- **安全最硬核**: OS 级沙箱 (Seatbelt/Landlock/Bwrap/RestrictedToken)，三平台原生实现
- **模块化最好**: ~50 Rust crates，边界清晰
- **Agent 批量能力**: CSV 批量生成 Agent — 三者中独有
- **流式优化**: WebSocket 预热 + HTTPS 降级
- **远程压缩**: 支持服务端压缩 (Azure 等)

## 12.4 架构权衡总结

| 权衡 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **简单 vs 灵活** | 单体更简单 | Server-Client 最灵活 | 模块化平衡 |
| **安全 vs 开放** | AI + 规则 + 沙箱 | 规则 + 用户确认 | OS 沙箱 + 策略 |
| **深度 vs 广度** | 6 Provider 深适配 | 15+ Provider 统一 | OpenAI 为主 |
| **内置 vs 扩展** | 40+ 工具内置 | 17 工具 + 插件 + MCP | 25+ 工具 + plugin/skills |
| **耦合 vs 解耦** | UI 和引擎同进程 | UI 和引擎分进程 | 独立 crates |
| **自研 vs 复用** | 自研 API/Ink/vim | 复用 AI SDK/Hono/Drizzle | 自研核心, 复用 infra |
| **JS vs Native** | TypeScript (Bun) | TypeScript (Bun) | Rust (tokio) |

## 12.5 总结

三个项目代表了 AI 编码助手的三种演化路径:

- **Claude Code** — Anthropic 自上而下的产品化路径: 控制一切、深度优化、安全优先
- **OpenCode** — 社区驱动的平台化路径: 开放生态、多端接入、Provider 无关
- **Codex** — OpenAI 性能导向的工程路径: Rust 原生、OS 级安全、模块化架构

没有绝对的优劣，选择取决于:
- 需要 Anthropic 深度集成 → Claude Code
- 需要多 Provider + 插件生态 → OpenCode
- 需要极致性能 + 硬核安全 → Codex
