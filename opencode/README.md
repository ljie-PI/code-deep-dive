# OpenCode 架构深度解析

> **版本**: 基于 opencode v1.4.6 (`dev` 分支, commit d7718d41d)
> **代码库**: https://github.com/anomalyco/opencode
> **语言**: TypeScript (Bun 运行时)
> **规模**: ~1421 个 TypeScript 文件, ~77,000 行核心包代码

---

## 阅读路径

### 初学者路线（从全局到细节）
1. [第一章：全局概览](./01-global-overview.md) — 系统是什么，有哪些部分
2. [第二章：核心循环](./02-core-loop.md) — 一条消息从输入到输出的完整旅程
3. [第三章：会话与消息模型](./03-session-message.md) — 数据如何组织和持久化
4. [第六章：工具系统](./06-tool-system.md) — Agent 如何与外部世界交互

### 开发者路线（快速定位）
1. [第一章：全局概览](./01-global-overview.md) — 模块地图
2. [第四章：Provider 与模型](./04-provider-model.md) — 如何接入 LLM
3. [第五章：Agent 配置](./05-agent-config.md) — Agent 定义与组合
4. [第九章：插件系统](./09-plugin-system.md) — 如何扩展 OpenCode

### 架构师路线（设计哲学）
1. [第一章：全局概览](./01-global-overview.md) — 架构全景
2. [第十章：Effect 依赖注入](./10-effect-di.md) — 为什么选 Effect，如何组织
3. [第十二章：设计哲学](./12-design-philosophy.md) — 核心设计决策

---

## 章节目录

| 章节 | 标题 | 核心主题 |
|------|------|---------|
| 01 | [全局概览](./01-global-overview.md) | 系统架构、模块划分、技术选型 |
| 02 | [核心循环](./02-core-loop.md) | Prompt → LLM → Tool → Loop 的完整流程 |
| 03 | [会话与消息模型](./03-session-message.md) | Session、Message、Part 数据模型与持久化 |
| 04 | [Provider 与模型](./04-provider-model.md) | 20+ LLM 提供商的统一抽象 |
| 05 | [Agent 配置](./05-agent-config.md) | Agent 定义、变体、权限 |
| 06 | [工具系统](./06-tool-system.md) | 19 个内置工具 + MCP 外部工具 |
| 07 | [权限系统](./07-permission-system.md) | 规则求值、安全门控 |
| 08 | [配置系统](./08-config-system.md) | 多层配置合并、Markdown 指令 |
| 09 | [插件系统](./09-plugin-system.md) | 插件加载、生命周期、扩展点 |
| 10 | [Effect 依赖注入](./10-effect-di.md) | Effect Service/Layer 模式 |
| 11 | [服务器与 API](./11-server-api.md) | Hono HTTP 服务器、WebSocket、SDK |
| 12 | [CLI 与终端 UI](./12-cli-tui.md) | yargs 命令、SolidJS TUI |
| 13 | [快照与版本控制](./13-snapshot-vcs.md) | Git 快照、回滚、差异 |
| 14 | [上下文管理](./14-context-management.md) | 压缩、摘要、令牌管理 |
| 15 | [外部协议集成](./15-protocols.md) | MCP、ACP、LSP |
| 16 | [Web 与桌面应用](./16-web-desktop.md) | SolidJS Web App、Tauri/Electron 桌面 |
| 17 | [存储与数据库](./17-storage-database.md) | SQLite、Drizzle ORM、事件总线 |
| 18 | [设计哲学与模式](./18-design-philosophy.md) | 关键设计决策与架构模式 |

---

## 附录

- [附录 A：关键文件索引](./appendix-a-file-index.md)
- [附录 B：术语表](./appendix-b-glossary.md)
- [附录 C：环境变量速查](./appendix-c-env-vars.md)

---

## 技术栈速览

| 领域 | 选择 |
|------|------|
| 运行时 | Bun 1.3.11 |
| 语言 | TypeScript 5.8.2 |
| 包管理 | Bun workspaces + Turborepo |
| 依赖注入 | Effect 4.0.0-beta |
| LLM SDK | Vercel AI SDK 6.x |
| HTTP 服务器 | Hono 4.x |
| 数据库 | SQLite (Drizzle ORM) |
| 终端 UI | SolidJS + opentui |
| Web 前端 | SolidJS + SolidStart |
| 桌面应用 | Tauri (Rust) + Electron |
| Schema | Zod 4.x |
| 工具协议 | MCP (Model Context Protocol) |
| Agent 协议 | ACP (Agent Client Protocol) |
