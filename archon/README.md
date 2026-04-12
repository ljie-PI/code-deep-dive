# Archon 架构深度解析

> Archon 是一个面向 AI 编码助手的工作流引擎。将开发流程定义为 YAML 工作流，通过 Slack、Telegram、GitHub、Discord、Gitea、GitLab、Web UI 或 CLI 触发执行。

## 阅读路径

### 快速了解（30 分钟）
1. [第一章：全局概览](01-overview.md) — 系统架构、包结构、核心循环
2. [第五章：工作流引擎](05-workflow-engine.md) §5.3-§5.4 — YAML 格式和节点类型
3. [第十一章：设计哲学](11-design-philosophy.md) §11.1-§11.2 — 核心设计决策

### 开发者入门（2 小时）
1. [第一章：全局概览](01-overview.md) — 建立心智模型
2. [第三章：基础层](03-foundation-layer.md) — 路径系统和 Git 操作
3. [第五章：工作流引擎](05-workflow-engine.md) — DAG 执行模型
4. [第六章：核心业务逻辑](06-core-business-logic.md) — handleMessage 流程
5. [第十一章：设计哲学](11-design-philosophy.md) — 扩展机制

### 完整阅读（4-6 小时）
按章节顺序阅读。

## 章节目录

| 章 | 标题 | 内容 |
|----|------|------|
| 01 | [全局概览](01-overview.md) | 系统定位、技术栈、Monorepo 结构、核心循环、数据流 |
| 02 | [数据模型与数据库](02-data-model.md) | 8 张表的 ER 图、数据库抽象层、迁移策略 |
| 03 | [基础层：Paths 与 Git](03-foundation-layer.md) | 路径解析、日志系统、CWD 清洗、Git 操作抽象 |
| 04 | [隔离系统](04-isolation-system.md) | Worktree 隔离、7 步解析流程、错误处理 |
| 05 | [工作流引擎](05-workflow-engine.md) | YAML DAG 格式、7 种节点类型、并发调度、变量替换 |
| 06 | [核心业务逻辑](06-core-business-logic.md) | handleMessage、AI 客户端、命令处理、Session 状态机 |
| 07 | [平台适配器](07-platform-adapters.md) | 6 个平台的接入方式、认证模式、消息分割 |
| 08 | [HTTP 服务器与 Web 适配器](08-server-web-adapter.md) | Hono + SSE、REST API、Webhook、消息持久化 |
| 09 | [命令行界面](09-cli.md) | CLI 命令结构、workflow run 流程、设置向导 |
| 10 | [Web 前端](10-web-frontend.md) | React SPA、SSE Hook、Zustand Store、工作流构建器 |
| 11 | [设计哲学与扩展机制](11-design-philosophy.md) | 设计原则、关键决策、安全设计、如何扩展 |

## 附录

- [关键文件索引](appendix-file-index.md) — 按重要性排序的文件清单
- [术语表](appendix-glossary.md) — Archon 特定术语定义

## 代码库信息

- **版本**：0.3.6
- **语言**：TypeScript (Bun)
- **总文件数**：289 个 `.ts` 文件
- **总代码行数**：~96,000 行
- **包数量**：10 个
- **数据库表**：8 个
- **分析日期**：2026-04-12
