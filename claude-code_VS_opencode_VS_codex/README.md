# Claude Code vs OpenCode vs Codex：架构深度对比

> **版本**: Claude Code v2.1.888 vs OpenCode v1.4.6 vs Codex (codex-rs)
> **对比维度**: 架构总览、核心循环、工具系统、权限安全、多Agent、上下文管理、配置系统、会话模型、MCP集成、终端UI、扩展机制、设计哲学
> **目标读者**: 想理解三种 AI Coding Agent 不同实现路径的架构师和开发者

---

## 目录

1. [总览对比](01-overview.md)
2. [核心循环 (Agent Loop)](02-agent-loop.md)
3. [工具系统](03-tool-system.md)
4. [权限与安全](04-permission-security.md)
5. [多 Agent 架构](05-multi-agent.md)
6. [上下文管理与压缩](06-context-management.md)
7. [配置系统](07-config-system.md)
8. [会话与消息模型](08-session-model.md)
9. [MCP 集成](09-mcp.md)
10. [终端 UI](10-terminal-ui.md)
11. [扩展机制](11-extension.md)
12. [设计哲学](12-design-philosophy.md)

---

## 一句话总结

| 项目 | 一句话 |
|------|--------|
| **Claude Code** | Anthropic 官方单体 TypeScript CLI，安全纵深防御、上下文管理精细、多Agent架构最复杂 |
| **OpenCode** | 社区开源 TypeScript+Effect 项目，Server-Client 架构、插件生态最强、Provider 支持最广 |
| **Codex** | OpenAI 官方 Rust 项目，OS 级沙箱隔离、性能与安全并重、模块化 crate 架构 |
