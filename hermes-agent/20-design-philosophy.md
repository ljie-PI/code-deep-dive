# 第二十章：设计哲学与扩展指南

> **一句话概括**：Hermes Agent 的架构围绕"技能优于工具、OpenAI 兼容优于定制协议、文件系统优于数据库、可组合性优于单体设计"的核心原则构建，为社区贡献和扩展提供了清晰的路径。

---

## 20.1 核心设计原则

### 原则 1：技能优于工具（Skills over Tools）

这是 Hermes 最根本的设计决策。CONTRIBUTING.md 明确指出：

> "Should it be a Skill or a Tool? The answer is almost always **skill**."

**为什么**：
- **工具**是 Python 代码，需要维护、测试、依赖管理，每次修改需要发版
- **技能**是 Markdown 指令，可以被代理即时加载、动态改进、社区分享
- 工具的能力边界是固定的；技能可以组合已有工具实现任意复杂的工作流

**判断标准**：

| 特征 | 做成技能 | 做成工具 |
|------|---------|---------|
| 可以用现有工具组合完成 | ✅ | |
| 需要自定义 Python 逻辑 | | ✅ |
| 需要处理二进制数据/流 | | ✅ |
| 包装外部 CLI/API | ✅ | |
| 需要 API 密钥管理 | | ✅ |
| 需要精确执行（非"尽力而为"） | | ✅ |

这个原则解释了为什么 Hermes 有 287 个技能但只有约 40 个工具——大多数能力被推到了技能层。

### 原则 2：OpenAI 兼容优于定制协议

Hermes 的 LLM 交互完全基于 OpenAI Chat Completions API 格式：

```python
# run_agent.py 使用 openai.OpenAI 客户端作为主接口
from openai import OpenAI
```

即使是 Anthropic 的 Claude，也通过 `agent/anthropic_adapter.py` 适配为 OpenAI 格式。这意味着：
- 任何提供 OpenAI 兼容端点的服务都可以即插即用
- 本地模型（Ollama、vLLM、llama.cpp server）零配置支持
- 不存在 provider lock-in

**唯一例外**：Anthropic 的 prompt caching（`agent/prompt_caching.py`）使用了原生 API 特性，但这是性能优化，不是核心功能。

### 原则 3：文件系统优于数据库

Hermes 的持久化策略偏好简单的文件系统存储：

| 数据 | 存储方式 | 原因 |
|------|---------|------|
| 技能 | Markdown 文件 (`~/.hermes/skills/`) | 人类可读、git 友好、可直接编辑 |
| 记忆 | Markdown 文件 (`~/.hermes/memory/`) | 同上 |
| 配置 | YAML (`~/.hermes/config.yaml`) | 人类可读 |
| 人格 | Markdown (`SOUL.md`) | 人类可读 |
| 上下文文件 | 任意文件 (`~/.hermes/context-files/`) | 零格式要求 |

**SQLite 仅用于需要查询的场景**：会话历史和 FTS5 全文搜索（`hermes_state.py`）。

这个选择使得：
- 用户可以直接编辑/备份/版本控制他们的代理数据
- 不需要数据库迁移工具
- 部署极其简单（复制 `~/.hermes/` 目录即可）

### 原则 4：可组合性优于单体设计

Hermes 的架构在多个层次上展现了组合性：

**工具层**：工具通过 `tools/registry.py` 自注册，工具集通过 `toolsets.py` 组合
```python
# 工具集可以包含其他工具集
TOOLSETS = {
    "full_stack": {
        "tools": [],
        "includes": ["web", "terminal", "files", "browser", ...]
    }
}
```

**平台层**：消息平台通过继承 `BasePlatform` 添加，互不干扰

**后端层**：终端后端通过 `tools/environments/base.py` 抽象，可以任意切换

**技能层**：技能可以调用其他技能，形成组合

### 原则 5：渐进式能力（Progressive Capability）

Hermes 的功能是分层可选的：

```
核心（无依赖）: CLI 对话 + 文件操作
  ↓ +messaging extra
消息平台: Telegram/Discord/Slack/...
  ↓ +voice extra  
语音: 语音转写 + TTS
  ↓ +mcp extra
MCP: 连接外部 MCP 服务器
  ↓ +modal/daytona extra
云端执行: 无服务器终端后端
  ↓ +rl extra
研究: RL 训练环境
```

这通过 `pyproject.toml` 的 optional dependencies 实现（`pyproject.toml:39-110`）。用户只安装他们需要的能力。

---

## 20.2 关键架构决策

### 决策 1：单文件核心循环

`run_agent.py`（10,524 行）是一个很大的文件。这是**有意为之**的：

- 对话循环的所有逻辑集中在一个地方，便于理解完整流程
- 避免了跨文件的控制流跳转
- 辅助功能（`agent/` 包）被提取出去，但核心循环保持内聚

这遵循了"关键路径应该是线性的"的设计哲学——开发者从 `run_conversation()` 开始，可以顺序阅读理解整个流程。

### 决策 2：工具自注册模式

```python
# tools/registry.py 提供 register() 函数
# 每个工具文件在 import 时自注册
from tools.registry import registry

registry.register(
    name="web_search",
    description="...",
    parameters={...},
    handler=handler_func,
    toolset="web",
)
```

**好处**：
- 添加工具只需创建一个文件并注册
- 不需要修改任何中央配置
- `model_tools.py` 的 `_discover_tools()` 自动扫描所有工具模块

### 决策 3：平台适配器模式

所有消息平台共享 `BasePlatform` 接口，但每个平台可以有完全不同的内部实现：

```
BasePlatform (gateway/platforms/base.py)
├── TelegramPlatform  — python-telegram-bot
├── DiscordPlatform   — discord.py
├── SlackPlatform     — slack-bolt
├── MatrixPlatform    — mautrix
├── APIServerPlatform — aiohttp REST API
└── ... 16+ more
```

每个平台处理自己的特殊性（消息格式、附件、反应、线程），网关核心只关心统一的消息流。

### 决策 4：辅助 LLM 客户端

`agent/auxiliary_client.py`（2,613 行）是一个专门为**非对话任务**设计的 LLM 客户端：

- 标题生成
- 上下文压缩/摘要
- 技能创建
- 记忆筛选

使用独立的客户端意味着：
- 这些任务不占用主对话的上下文窗口
- 可以使用更便宜/更快的模型
- 失败不影响主对话

### 决策 5：SQLite 状态存储

`hermes_state.py` 使用 SQLite WAL 模式：

- **WAL 模式**允许并发读写（网关多平台场景）
- **FTS5** 提供全文搜索，支持跨会话回忆
- **Session splitting** 通过 `parent_session_id` 链处理压缩后的会话延续

这比 JSONL 文件方案提供了更好的查询能力，同时保持了嵌入式数据库的简单部署特性。

---

## 20.3 扩展指南

### 添加新工具

1. 在 `tools/` 目录创建新文件
2. 实现 handler 函数
3. 调用 `registry.register()` 注册
4. 在 `toolsets.py` 的相应工具集中添加工具名
5. 如需审批，在 `tools/approval.py` 添加规则
6. 编写测试

### 添加新消息平台

1. 阅读 `gateway/platforms/ADDING_A_PLATFORM.md`
2. 继承 `BasePlatform`（`gateway/platforms/base.py`）
3. 实现必需的方法：消息接收、消息发送、平台特性
4. 在 `gateway/run.py` 中注册
5. 在 `hermes_cli/gateway.py` 中添加配置命令

### 添加新终端后端

1. 继承 `tools/environments/base.py` 中的基类
2. 实现命令执行、文件操作等方法
3. 在 `tools/terminal_tool.py` 中注册

### 创建新技能

1. 在 `skills/[category]/` 目录创建 Markdown 文件
2. 使用标准格式：标题 + 描述 + 指令
3. 或通过代理自动创建（`skill_manage` 工具）
4. 发布到 Skills Hub 进行社区分享

### 添加新 Provider

1. 如果提供 OpenAI 兼容 API：只需配置 base_url 和 API key
2. 如果使用原生 SDK：参考 `agent/anthropic_adapter.py` 编写适配器
3. 在 `hermes_cli/providers.py` 添加 provider 定义
4. 在 `hermes_cli/models.py` 添加模型列表

---

## 20.4 架构演进方向

基于代码中的线索（TODO 注释、optional dependencies、实验性功能），可以观察到以下演进趋势：

1. **ACP 集成深化**：`acp_adapter/` 正在发展为编辑器集成的标准接口
2. **MCP 生态扩展**：既是 MCP 客户端（连接外部工具）又是 MCP 服务器（暴露自身工具）
3. **RL 训练闭环**：`environments/` 和 `batch_runner.py` 表明正在构建从使用数据到模型改进的完整闭环
4. **更多消息平台**：已支持 16+ 平台，持续扩展中（飞书、钉钉、企业微信等中国平台）
5. **Honcho 用户建模**：从简单记忆向辩证式用户理解演进

---

## 20.5 代码质量特征

### 测试覆盖
- 176,179 行测试代码 vs 193,682 行源代码（≈0.91 测试/源比率）
- 使用 pytest + pytest-asyncio + pytest-xdist（并行测试）
- 集成测试标记分离：`@pytest.mark.integration`

### 错误处理模式
- API 错误通过 `error_classifier.py` 分类，支持自动 failover
- 重试通过 `jittered_backoff` 实现（指数退避 + 随机抖动）
- stdio 通过 `_SafeWriter` 包装，防止管道断裂导致崩溃

### 安全考量
- 工具审批系统防止危险命令自动执行
- 路径安全检查防止目录遍历
- 凭证文件检测防止敏感信息泄露
- URL 安全验证防止 SSRF
- 依赖版本固定在已知安全范围（`pyproject.toml` 注释中标注了 CVE）

---

## 20.6 关键文件索引

| 文件 | 行数 | 职责 | 扩展相关性 |
|------|------|------|-----------|
| `run_agent.py` | 10,524 | 核心对话循环 | 低 — 通常不需要修改 |
| `tools/registry.py` | — | 工具注册表 | 高 — 添加工具的入口 |
| `toolsets.py` | 655 | 工具集定义 | 高 — 添加工具后需要更新 |
| `gateway/platforms/base.py` | 1,998 | 平台基类 | 高 — 添加平台的接口 |
| `tools/environments/base.py` | — | 后端基类 | 高 — 添加后端的接口 |
| `agent/prompt_builder.py` | — | 提示词构建 | 中 — 定制提示词时需要 |
| `hermes_cli/config.py` | 3,095 | 配置管理 | 中 — 添加配置项时需要 |
| `tools/approval.py` | — | 审批系统 | 中 — 添加安全规则时需要 |
