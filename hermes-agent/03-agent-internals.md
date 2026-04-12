# 第三章：Agent 内部机制

> **一句话概述：** `agent/` 包是 Hermes Agent 核心循环的支撑基础设施层，负责系统提示词组装、上下文压缩、模型元数据管理、错误分类与恢复、凭证轮换、辅助 LLM 调用、用量追踪以及终端呈现——所有逻辑均以无状态函数或独立类的形式从 `run_agent.py` 中剥离，使主编排器 `AIAgent` 能够专注于对话循环本身。

---

## 3.1 架构总览

```mermaid
%%{init: {'theme': 'neutral'}}%%
graph TB
    subgraph run_agent["run_agent.py (AIAgent 编排器)"]
        LOOP["主对话循环"]
    end

    subgraph prompt["提示词构建层"]
        PB["prompt_builder.py<br/>系统提示词组装"]
        MM["memory_manager.py<br/>记忆编排"]
        MP["memory_provider.py<br/>记忆提供者接口"]
        SC["skill_commands.py<br/>技能斜杠命令"]
        SU["skill_utils.py<br/>技能元数据工具"]
    end

    subgraph context["上下文管理层"]
        CE["context_engine.py<br/>上下文引擎基类"]
        CC["context_compressor.py<br/>有损摘要压缩"]
        CR["context_references.py<br/>@ 引用展开"]
        SH["subdirectory_hints.py<br/>子目录提示发现"]
        MCF["manual_compression_feedback.py"]
    end

    subgraph model["模型抽象层"]
        MD["model_metadata.py<br/>上下文长度 & token 估算"]
        SR["smart_model_routing.py<br/>廉价/强力模型路由"]
        AA["anthropic_adapter.py<br/>Anthropic 原生适配"]
        AC["auxiliary_client.py<br/>辅助 LLM 路由"]
        CP["credential_pool.py<br/>凭证池化与轮换"]
        CAC["copilot_acp_client.py<br/>GitHub Copilot ACP"]
        PC["prompt_caching.py<br/>Anthropic 缓存控制"]
    end

    subgraph error["错误处理层"]
        EC["error_classifier.py<br/>错误分类管线"]
        RU["retry_utils.py<br/>抖动退避"]
        RL["rate_limit_tracker.py<br/>速率限制追踪"]
    end

    subgraph persist["持久化层"]
        TR["trajectory.py<br/>对话轨迹序列化"]
        UP["usage_pricing.py<br/>用量 & 成本估算"]
        IN["insights.py<br/>会话分析洞察"]
    end

    subgraph display["显示层"]
        DP["display.py<br/>终端呈现"]
        TG["title_generator.py<br/>自动标题生成"]
    end

    subgraph security["安全层"]
        RD["redact.py<br/>敏感数据脱敏"]
    end

    LOOP --> PB
    LOOP --> CC
    LOOP --> MD
    LOOP --> EC
    LOOP --> DP
    LOOP --> TR
    LOOP --> RD

    PB --> MM
    PB --> SU
    MM --> MP
    CC --> CE
    CC --> AC
    CC --> MD
    AC --> CP
    EC --> RU
    IN --> UP
    TG --> AC
```

`run_agent.py` 中的 `AIAgent` 类是整个系统的编排器。它不直接实现提示词拼接、压缩或错误恢复的细节，而是通过调用 `agent/` 包中的各个模块来完成这些工作。`agent/__init__.py` 的注释一针见血地说明了设计意图：

> *"These modules contain pure utility functions and self-contained classes that were previously embedded in the 3,600-line run_agent.py."*（`agent/__init__.py:1-6`）

---

## 3.2 提示词构建（Prompt Construction）

### 3.2.1 prompt_builder.py — 系统提示词的中枢

**用途：** 组装发送给 LLM 的系统提示词。所有函数均为无状态纯函数，由 `AIAgent._build_system_prompt()` 调用后合并。

**核心常量：**

| 常量 | 行号 | 用途 |
|------|------|------|
| `DEFAULT_AGENT_IDENTITY` | :134 | 默认 Agent 身份描述（"You are Hermes Agent..."） |
| `MEMORY_GUIDANCE` | :144 | 指导 Agent 如何使用持久记忆 |
| `SKILLS_GUIDANCE` | :164 | 指导 Agent 何时创建/更新 Skill |
| `TOOL_USE_ENFORCEMENT_GUIDANCE` | :173 | 强制工具使用（防止 Agent 只描述不执行） |
| `OPENAI_MODEL_EXECUTION_GUIDANCE` | :196 | GPT/Codex 特定执行纪律（含 6 个 XML 块） |
| `GOOGLE_MODEL_OPERATIONAL_GUIDANCE` | :258 | Gemini/Gemma 操作指令 |
| `PLATFORM_HINTS` | :285 | 12 种平台适配提示（WhatsApp/Telegram/Discord/Slack 等） |
| `DEVELOPER_ROLE_MODELS` | :283 | 使用 `developer` 角色而非 `system` 的模型列表 |

**关键函数：**

- `build_context_files_prompt(cwd, skip_soul) -> str`（:986）：按优先级发现并加载项目上下文文件。优先级链为 `.hermes.md` > `AGENTS.md` > `CLAUDE.md` > `.cursorrules`，首个匹配生效。
- `build_skills_system_prompt(available_tools, available_toolsets) -> str`（:563）：构建技能索引。采用**两层缓存架构**——进程内 LRU 字典 + 磁盘快照（`.skills_prompt_snapshot.json`，以 mtime/size 清单校验）。
- `load_soul_md() -> Optional[str]`（:873）：从 `HERMES_HOME/SOUL.md` 加载自定义 Agent 身份。
- `build_environment_hints() -> str`（:387）：检测 WSL 等执行环境并生成提示。

**安全机制：** 所有上下文文件在注入系统提示词前都要经过 `_scan_context_content()` 扫描（:55），它检测 10 类提示注入模式（如 `ignore previous instructions`、隐藏 HTML div、curl 窃取凭证等）以及不可见 Unicode 字符，若发现威胁则整个文件被阻止加载。

**截断策略：** `_truncate_content()` 对超过 20,000 字符的文件执行头尾截断（70% 头部 + 20% 尾部），中间插入截断标记。

### 3.2.2 memory_manager.py — 记忆编排器

**用途：** 统一管理内置记忆（`MEMORY.md`/`USER.md`）和外部记忆插件（Honcho、Hindsight 等），是 `run_agent.py` 中记忆相关逻辑的唯一集成点。

**核心类：** `MemoryManager`（:72）

**设计约束：** 内置 Provider 始终注册且不可移除；最多允许一个外部 Provider（:86-108），拒绝注册第二个外部 Provider 并输出警告。

**生命周期方法：**

```python
class MemoryManager:
    def add_provider(self, provider: MemoryProvider) -> None    # 注册 Provider
    def build_system_prompt(self) -> str                        # 收集系统提示词块
    def prefetch_all(self, query, session_id) -> str            # 查询前预取
    def sync_all(self, user_content, assistant_content) -> None # 轮次后同步
    def handle_tool_call(self, tool_name, args) -> str          # 路由工具调用
    def on_pre_compress(self, messages) -> str                  # 压缩前通知
    def shutdown_all(self) -> None                              # 反序关闭
```

**上下文围栏（Context Fencing）：** `build_memory_context_block()`（:54）将预取到的记忆包裹在 `<memory-context>` 标签内，并附加系统注释说明这是"回忆的记忆上下文，不是新的用户输入"，防止 LLM 将召回内容误解为用户对话。

### 3.2.3 memory_provider.py — 记忆提供者抽象

**用途：** 定义所有记忆 Provider 必须实现的 `MemoryProvider` ABC。

**抽象方法：** `name`（属性）、`is_available()`、`initialize()`、`get_tool_schemas()`

**可选钩子：** `on_turn_start()`、`on_session_end()`、`on_pre_compress()`、`on_delegation()`、`on_memory_write()`

钩子中 `on_delegation()` 值得关注——它在**父 Agent** 的 Provider 上被调用，接收子 Agent 的任务和结果作为观测事件，使记忆系统能追踪委派工作。

### 3.2.4 skill_utils.py / skill_commands.py — 技能工具

**skill_utils.py**（:1）提供轻量级技能元数据工具：前置声明解析（`parse_frontmatter`）、平台兼容性检查（`skill_matches_platform`）、技能条件提取（`extract_skill_conditions`）等。该模块故意避免导入工具注册表或 CLI 配置等重型依赖。

**skill_commands.py**（:1）提供在 CLI 和网关之间共享的斜杠命令助手，如 `/plan` 路径生成（`build_plan_path()`，:24）和技能加载（`_load_skill_payload()`，:45）。

---

## 3.3 上下文管理（Context Management）

### 3.3.1 context_engine.py — 可插拔上下文引擎基类

**用途：** 定义上下文引擎的抽象接口。内置的 `ContextCompressor` 是默认实现，第三方引擎（如 LCM）可通过插件系统替换，由 `context.engine` 配置项驱动。

**核心抽象方法：**

```python
class ContextEngine(ABC):
    def update_from_response(self, usage: Dict) -> None        # 更新 token 用量
    def should_compress(self, prompt_tokens: int) -> bool       # 判断是否需压缩
    def compress(self, messages, current_tokens) -> List[Dict]  # 执行压缩
```

**属性约定：** 引擎必须维护 `last_prompt_tokens`、`threshold_tokens`、`context_length`、`compression_count` 等属性，`run_agent.py` 直接读取它们进行显示和日志记录（:46-51）。

### 3.3.2 context_compressor.py — 有损摘要压缩器

**用途：** 当对话上下文接近模型 token 上限时，通过 LLM 摘要中间轮次来释放空间。这是 Hermes Agent 最复杂的上下文管理机制。

**核心类：** `ContextCompressor(ContextEngine)`（:60）

**压缩算法（5 阶段）：**

```
阶段 1: 工具输出修剪（廉价预处理，无 LLM 调用）
   └── _prune_old_tool_results() 将超过 200 字符的旧工具结果替换为占位符

阶段 2: 确定保护边界
   └── 保护头部消息（系统提示 + 首次交换，默认 3 条）
   └── 按 token 预算保护尾部（约 20K token 的最近上下文）

阶段 3: LLM 结构化摘要
   └── _generate_summary() 使用辅助 LLM 生成结构化摘要

阶段 4: 组装压缩后的消息列表
   └── 摘要以 SUMMARY_PREFIX 前缀标记为"移交参考"

阶段 5: 工具调用对完整性修复
   └── _sanitize_tool_pairs() 清理孤立的 tool_call/tool_result
```

**关键设计决策：**

1. **token 预算尾部保护**（:604）：不使用固定消息数，而是从尾部反向累积 token 直到预算用尽，由 `tail_token_budget`（= `threshold_tokens * summary_target_ratio`）控制。这意味着大上下文模型自动获得更多尾部保护。

2. **迭代摘要更新**（:406）：后续压缩不从零开始，而是基于 `_previous_summary` 进行增量更新。提示词明确指示"保留现有信息，添加新进展，将完成项从进行中移至已完成"。

3. **结构化摘要模板**：包含 Goal、Progress（Done/In Progress/Blocked）、Key Decisions、Resolved Questions、Pending User Asks、Relevant Files、Remaining Work、Critical Context、Tools & Patterns 九个章节。

4. **角色交替保障**（:769-800）：摘要消息的角色经过精心选择以避免连续相同角色。若两个角色都会导致冲突，则将摘要合并到第一条尾部消息中。

5. **聚焦压缩**（`focus_topic` 参数，:437）：受 Claude Code 的 `/compact` 启发，用户可指定关注主题，摘要器将 60-70% 的预算分配给相关内容。

6. **失败冷却**（:469）：摘要生成失败后进入 600 秒冷却期，期间直接丢弃中间轮次而不尝试摘要，防止重复失败。

### 3.3.3 context_references.py — @ 引用系统

**用途：** 解析和展开用户消息中的 `@` 引用语法，将外部内容注入对话上下文。

**支持的引用类型：**

| 语法 | 类型 | 处理方式 |
|------|------|----------|
| `@file:path` | 文件 | 读取内容，支持行范围 `:start-end` |
| `@folder:path` | 文件夹 | 构建目录树（利用 `rg --files`） |
| `@diff` | Git diff | 执行 `git diff` |
| `@staged` | 暂存区 | 执行 `git diff --staged` |
| `@git:N` | Git 日志 | 执行 `git log -N -p` |
| `@url:URL` | 网页 | 通过 `web_extract_tool` 获取 |

**安全控制：**
- **注入限制**：硬限制为上下文长度的 50%，超过则拒绝注入（:169）；软限制为 25%，超过则发出警告。
- **敏感路径阻止**（:342-360）：阻止访问 `.ssh`、`.aws`、`.gnupg` 等目录以及 `.env`、`credentials` 等文件。
- **路径沙箱**（:329-339）：默认约束在 CWD 内，防止 `@` 引用逃逸到工作区之外。

### 3.3.4 subdirectory_hints.py — 渐进式子目录发现

**用途：** 当 Agent 通过工具调用导航到子目录时，自动发现并加载该目录下的上下文文件（`AGENTS.md`、`CLAUDE.md`、`.cursorrules`）。

**核心类：** `SubdirectoryHintTracker`（:48）

**工作机制：** 追踪器维护一个已加载目录集合 `_loaded_dirs`。每次工具调用后，`check_tool_call()` 从工具参数中提取路径（直接路径参数或从 shell 命令中解析），然后向上遍历最多 5 级祖先目录（:129），在每个新发现的目录中查找上下文文件。发现的提示被追加到工具结果中，无需修改系统提示词（保持提示缓存有效性）。

---

## 3.4 模型抽象层（Model Abstraction）

### 3.4.1 model_metadata.py — 模型能力检测与上下文长度管理

**用途：** 纯工具模块，提供上下文长度检测、token 估算和本地服务器识别。被 `ContextCompressor` 和 `run_agent.py` 广泛使用。

**上下文长度解析链（10 层优先级，:917-1044）：**

```
0. 用户配置覆盖 (config.yaml model.context_length)
1. 持久缓存 (context_length_cache.yaml)
2. 自定义端点 /models API
3. 本地服务器直接查询 (Ollama/LM Studio/vLLM/llama.cpp)
4. Anthropic /v1/models API
5. 提供者感知查找 (models.dev)
6. OpenRouter 实时 API
7. models.dev 注册表 (provider-aware)
8. 硬编码默认值 (DEFAULT_CONTEXT_LENGTHS)
9. 最终回退 (128K)
```

**本地服务器检测：** `detect_local_server_type(base_url) -> str`（:294）通过探测特定端点识别四种本地服务器：Ollama（`/api/tags`）、LM Studio（`/api/v1/models`）、llama.cpp（`/v1/props`）、vLLM（`/version`）。

**上下文探测阶层（:77-83）：** 当模型未知时，从 128K 开始逐级下探：`128K → 64K → 32K → 16K → 8K`。

**token 估算：** `estimate_tokens_rough(text) -> int`（:1047）使用 ~4 字符/token 的粗略估计（ceiling 除法保证短文本不估为 0）。`estimate_request_tokens_rough()` 还包含工具 schema 估算——50+ 工具的 schema 可占 20-30K token。

**提供者前缀剥离：** `_strip_provider_prefix()`（:48）能区分 `"local:my-model"` → `"my-model"`（提供者前缀）和 `"qwen3.5:27b"` → 保持不变（Ollama model:tag），通过正则匹配 Ollama 标签模式实现。

### 3.4.2 smart_model_routing.py — 动态模型路由

**用途：** 根据用户消息的复杂度，在廉价模型和主力模型之间动态切换。

**核心函数：** `choose_cheap_model_route(user_message, routing_config) -> Optional[Dict]`（:62）

**路由判定算法（保守策略——有疑则用主模型）：**

```
消息长度 > 160 字符? → 主模型
词数 > 28? → 主模型
换行 > 1? → 主模型
包含代码围栏? → 主模型
包含 URL? → 主模型
包含复杂关键词 (debug/implement/refactor/architecture 等 ~35 个)? → 主模型
以上均否 → 廉价模型
```

`resolve_turn_route()`（:110）组合路由决策和运行时提供者解析，返回包含 model/runtime/signature/label 的完整路由描述。

### 3.4.3 anthropic_adapter.py — Anthropic 原生 API 适配

**用途：** 在 Hermes 内部的 OpenAI 格式消息和 Anthropic Messages API 之间进行双向转换。

**关键特性：**
- **三种认证方式**：常规 API key（`sk-ant-api*`）→ `x-api-key` 头；OAuth token（`sk-ant-oat*`）→ Bearer 认证 + beta 头；Claude Code 凭证 → Bearer 认证。
- **输出 token 限制表**（:43-65）：为每个 Claude 模型精确配置最大输出 token（如 `claude-opus-4-6: 128K`，`claude-3-opus: 4096`）。
- **思考（Thinking）预算**（:30）：支持 4 档预算——xhigh(32K)、high(16K)、medium(8K)、low(4K)。
- **自适应推理努力**（:31）：映射到 Anthropic 的 adaptive effort 级别。

### 3.4.4 auxiliary_client.py — 辅助 LLM 客户端

**用途：** 为侧任务（上下文压缩、会话搜索、网页提取、视觉分析、标题生成等）提供统一的 LLM 调用入口。

**文本任务自动检测链（7 级优先级，:1-38）：**

```
1. OpenRouter (OPENROUTER_API_KEY)
2. Nous Portal (~/.hermes/auth.json)
3. 自定义端点 (config.yaml)
4. Codex OAuth (Responses API via chatgpt.com)
5. Native Anthropic
6. 直接 API key 提供者 (Z.AI/GLM, Kimi, MiniMax 等)
7. None
```

**视觉任务检测链**有不同的优先级顺序。

每个任务可在 `config.yaml` 的 `auxiliary:` 段覆盖提供者/模型/超时设置（如 `auxiliary.vision.provider`、`auxiliary.compression.model`）。

**支付耗尽容错：** 当解析的提供者返回 HTTP 402，`call_llm()` 自动用下一个可用提供者重试。

### 3.4.5 credential_pool.py — 凭证池化与轮换

**用途：** 管理同一提供者的多个 API 凭证，实现 429/402 时的自动轮换。

**`PooledCredential` 数据类**（:93）记录每个凭证的 provider、优先级、状态、访问令牌、使用计数和冷却时间。

**轮换策略（4 种，:64-70）：**

| 策略 | 说明 |
|------|------|
| `fill_first` | 优先用尽一个凭证再切换 |
| `round_robin` | 轮流使用 |
| `random` | 随机选择 |
| `least_used` | 使用次数最少者优先 |

**冷却机制：** 429 和 402 错误触发 1 小时冷却期（:74-76），冷却期间跳过该凭证。提供者返回的 `reset_at` 时间戳会覆盖默认冷却。

### 3.4.6 prompt_caching.py — Anthropic 提示缓存

**用途：** 通过缓存对话前缀，减少约 75% 的 Anthropic 输入 token 成本。

**策略：** `system_and_3`——使用 Anthropic 最多允许的 4 个 `cache_control` 断点：

1. 系统提示词（跨所有轮次稳定）
2-4. 最后 3 条非系统消息（滑动窗口）

`apply_anthropic_cache_control()`（:41）深拷贝消息列表后注入 `{"type": "ephemeral"}` 标记（支持 `5m` 和 `1h` TTL），处理 string content 和 list content 两种格式。

### 3.4.7 copilot_acp_client.py — GitHub Copilot ACP 适配

**用途：** 将 Hermes 请求转发到 `copilot --acp` 进程（JSON-RPC over stdio），使 GitHub Copilot 可作为聊天后端使用。

每个请求启动一个短生命周期的 ACP 会话，将格式化对话作为单个提示发送，收集文本块后转换回 Hermes 期望的 OpenAI 客户端形状。支持从响应中提取 `<tool_call>` XML 块以还原工具调用。

---

## 3.5 错误处理与恢复（Error Handling）

### 3.5.1 error_classifier.py — API 错误分类管线

**用途：** 用结构化分类替代分散的内联字符串匹配，为 `run_agent.py` 的重试循环提供统一的错误恢复建议。

**错误分类枚举 `FailoverReason`**（:25-58）定义了 13 种失败原因：

| 原因 | 恢复动作 |
|------|----------|
| `auth` / `auth_permanent` | 刷新/轮换 凭证，或中止 |
| `billing` | 立即轮换凭证 |
| `rate_limit` | 退避后轮换 |
| `overloaded` / `server_error` | 退避重试 |
| `timeout` | 重建客户端 + 重试 |
| `context_overflow` | **压缩**（不是故障转移） |
| `model_not_found` | 回退到其他模型 |
| `thinking_signature` | Anthropic thinking 块签名无效 |
| `long_context_tier` | Anthropic 超长上下文层级门控 |

**分类管线**（`classify_api_error()`，:222）按优先级执行：

```
1. 提供者特定模式（thinking 签名、层级门控）
2. HTTP 状态码 + 消息感知细化
3. 错误码分类（从响应体）
4. 消息模式匹配（计费/限速/上下文/认证）
5. 传输错误启发式
6. 服务器断连 + 大会话 → 上下文溢出
7. 兜底：unknown（可重试 + 退避）
```

**`ClassifiedError` 数据类**（:63）携带结构化恢复提示：`retryable`、`should_compress`、`should_rotate_credential`、`should_fallback`，重试循环直接检查这些标志而无需重新分类。

**OpenRouter 元数据解包**（:267-296）：OpenRouter 常将上游错误包裹在 `{"error": {"metadata": {"raw": "..."}}}` 中，分类器会递归解包到内层 JSON。

### 3.5.2 retry_utils.py — 抖动退避

**用途：** 以去相关的抖动延迟替代固定指数退避，防止并发会话的雷鸣群效应。

```python
def jittered_backoff(attempt, *, base_delay=5.0, max_delay=120.0, jitter_ratio=0.5) -> float
```

**算法：** `delay = min(base * 2^(attempt-1), max) + uniform(0, jitter_ratio * delay)`

抖动种子使用 `time_ns() ^ (counter * 0x9E3779B9)` 以确保即使在粗粒度时钟下也能去相关化（:53）。

### 3.5.3 rate_limit_tracker.py — 速率限制追踪

**用途：** 捕获提供者响应中的 `x-ratelimit-*` 头（12 个头字段），解析为结构化 `RateLimitState`，供 `/usage` 命令显示。

**数据模型：**
- `RateLimitBucket`（:31）：一个限速窗口（limit/remaining/reset_seconds/captured_at）
- `RateLimitState`（:57）：四个桶（requests_min, requests_hour, tokens_min, tokens_hour）

**显示功能：** `format_rate_limit_display()` 生成带 ASCII 进度条的终端输出，`format_rate_limit_compact()` 生成单行摘要用于状态栏。当任何桶超过 80% 使用率时自动附加警告。

---

## 3.6 持久化（Persistence）

### 3.6.1 trajectory.py — 对话轨迹序列化

**用途：** 将对话以 ShareGPT 格式追加到 JSONL 文件，用于训练数据收集或调试。

**关键函数：**
- `save_trajectory(trajectory, model, completed, filename)`（:30）：追加一条包含时间戳和完成状态的 JSONL 记录。
- `convert_scratchpad_to_think(content)`（:15）：将 `<REASONING_SCRATCHPAD>` 标签转换为 `<think>` 标签。

### 3.6.2 usage_pricing.py — 用量追踪与成本估算

**用途：** 规范化来自不同提供者的 token 用量数据，并基于定价信息估算 USD 成本。

**核心数据类：**
- `CanonicalUsage`（:28）：规范化用量（input/output/cache_read/cache_write/reasoning tokens）
- `BillingRoute`（:47）：计费路径（provider + model + base_url + billing_mode）
- `PricingEntry`（:55）：定价信息（input/output/cache_read/cache_write cost per million）
- `CostResult`（:67）：成本结果（amount_usd + status + source + notes）

**成本状态区分：** `CostStatus` 类型区分 `actual`（实际）、`estimated`（估算）、`included`（含在套餐中）、`unknown`。

### 3.6.3 insights.py — 会话分析洞察引擎

**用途：** 分析 SQLite 状态数据库中的历史会话数据，生成综合用量洞察——token 消耗、成本估算、工具使用模式、活跃趋势、模型/平台分布和会话指标。

**核心类：** `InsightsEngine`，提供 `generate(days=30)` 和 `format_terminal(report)` 方法。灵感来自 Claude Code 的 `/insights` 命令，但增加了多平台架构的成本估算和平台分析能力。

---

## 3.7 显示层（Display）

### 3.7.1 display.py — 终端呈现

**用途：** 提供 CLI 反馈相关的纯显示功能——旋转器、diff 预览、工具调用格式化。被 `AIAgent._execute_tool_calls` 使用。

**关键组件：**
- `LocalEditSnapshot`（:97）：工具执行前的文件系统快照，用于在写入后渲染本地 diff。
- **皮肤感知 diff 着色**（:32-77）：从活跃皮肤引擎懒加载 ANSI 颜色，支持亮/暗主题自适应。`_diff_ansi()` 从皮肤获取 `banner_dim`、`session_label`、`session_border`、`ui_error`、`ui_ok` 颜色并转换为 24-bit ANSI 转义。

### 3.7.2 title_generator.py — 自动标题生成

**用途：** 从首次用户/助手交换中自动生成 3-7 词的会话标题。在首次回复交付后异步运行，不增加用户感知延迟。

**工作流程：**
1. `maybe_auto_title()`（:95）检查是否为前 2 次交换且无标题
2. 在守护线程中调用 `auto_title_session()`
3. 使用辅助 LLM（最廉价/最快的可用模型）生成标题
4. 清理引号、前缀、截断至 80 字符后存入 session DB

---

## 3.8 安全（Security）

### 3.8.1 redact.py — 敏感数据脱敏

**用途：** 在日志、工具输出和网关日志中应用正则匹配以遮蔽 API key、token 和凭证。

**脱敏规则（8 类模式）：**

| 类别 | 模式数 | 示例 |
|------|--------|------|
| 已知前缀 key | 26 种 | `sk-*`、`ghp_*`、`AIza*`、`hf_*`、`AKIA*` 等 |
| 环境变量赋值 | 1 种 | `OPENAI_API_KEY=sk-abc...` |
| JSON 字段 | 1 种 | `"apiKey": "..."` |
| Authorization 头 | 1 种 | `Authorization: Bearer ...` |
| Telegram bot token | 1 种 | `bot1234567890:AAH...` |
| 私钥块 | 1 种 | `-----BEGIN RSA PRIVATE KEY-----` |
| 数据库连接串 | 1 种 | `postgres://user:pass@host` |
| E.164 电话号码 | 1 种 | `+12345678901` |

**遮蔽策略（:106-110）：** 短 token（< 18 字符）完全遮蔽为 `***`；长 token 保留前 6 + 后 4 字符以便调试（`sk-ant...xyz9`）。

**防篡改：** `_REDACT_ENABLED` 在**导入时快照**环境变量（:18），运行时的 `export HERMES_REDACT_SECRETS=false` 无法禁用脱敏。

**日志集成：** `RedactingFormatter(logging.Formatter)`（:173）在格式化后自动应用脱敏，作为日志处理器挂载使用。

---

## 3.9 关键文件索引

| 文件 | 行数 | 核心导出 | 主要调用者 |
|------|------|----------|------------|
| `prompt_builder.py` | 1026 | `build_context_files_prompt()`, `build_skills_system_prompt()`, `load_soul_md()`, `DEFAULT_AGENT_IDENTITY` | `AIAgent._build_system_prompt()` |
| `context_compressor.py` | 821 | `ContextCompressor` | `AIAgent` 主循环 |
| `memory_manager.py` | 363 | `MemoryManager`, `build_memory_context_block()` | `run_agent.py` |
| `memory_provider.py` | 232 | `MemoryProvider` (ABC) | 内置 + 插件 Provider |
| `model_metadata.py` | 1086 | `get_model_context_length()`, `estimate_tokens_rough()`, `detect_local_server_type()` | 压缩器、run_agent、辅助客户端 |
| `smart_model_routing.py` | 196 | `resolve_turn_route()`, `choose_cheap_model_route()` | `AIAgent` 每轮路由 |
| `auxiliary_client.py` | ~2800 | `call_llm()`, `resolve_provider_client()` | 压缩器、标题生成器、视觉分析 |
| `context_engine.py` | 185 | `ContextEngine` (ABC) | 压缩器、LCM 插件 |
| `context_references.py` | 521 | `preprocess_context_references()`, `parse_context_references()` | 用户消息预处理 |
| `prompt_caching.py` | 73 | `apply_anthropic_cache_control()` | Anthropic 调用路径 |
| `error_classifier.py` | ~400 | `classify_api_error()`, `FailoverReason`, `ClassifiedError` | `AIAgent` 重试循环 |
| `retry_utils.py` | 58 | `jittered_backoff()` | 重试循环 |
| `credential_pool.py` | ~1400 | `PooledCredential`, `load_pool()` | 凭证轮换逻辑 |
| `rate_limit_tracker.py` | 247 | `parse_rate_limit_headers()`, `format_rate_limit_display()` | `/usage` 命令 |
| `trajectory.py` | 57 | `save_trajectory()`, `convert_scratchpad_to_think()` | 批量运行器 |
| `usage_pricing.py` | ~600 | `estimate_usage_cost()`, `CanonicalUsage`, `normalize_usage()` | insights、/usage 命令 |
| `display.py` | ~1000 | `LocalEditSnapshot`, diff 渲染函数 | `AIAgent._execute_tool_calls` |
| `insights.py` | ~850 | `InsightsEngine` | `/insights` 命令 |
| `title_generator.py` | 126 | `maybe_auto_title()`, `generate_title()` | 首次交换后 |
| `skill_commands.py` | ~350 | `build_plan_path()`, 技能斜杠命令 | CLI + 网关 |
| `skill_utils.py` | ~400 | `parse_frontmatter()`, `skill_matches_platform()` | prompt_builder |
| `subdirectory_hints.py` | 225 | `SubdirectoryHintTracker` | `AIAgent` 工具执行 |
| `redact.py` | 182 | `redact_sensitive_text()`, `RedactingFormatter` | 日志系统 |
| `anthropic_adapter.py` | ~1400 | `build_anthropic_client()`, `build_anthropic_kwargs()` | Anthropic 调用路径 |
| `copilot_acp_client.py` | ~500 | `CopilotACPClient` | Copilot ACP 模式 |

---

## 3.10 跨组件交互模式总结

```mermaid
%%{init: {'theme': 'neutral'}}%%
sequenceDiagram
    participant User
    participant AIAgent as AIAgent<br/>(run_agent.py)
    participant PB as prompt_builder
    participant MM as MemoryManager
    participant CC as ContextCompressor
    participant EC as ErrorClassifier
    participant AC as AuxiliaryClient
    participant CP as CredentialPool

    User->>AIAgent: 发送消息
    AIAgent->>PB: build_context_files_prompt()
    AIAgent->>PB: build_skills_system_prompt()
    AIAgent->>MM: prefetch_all(query)
    AIAgent->>MM: build_memory_context_block()
    
    Note over AIAgent: 组装完整系统提示词
    
    AIAgent->>CC: should_compress(prompt_tokens)?
    alt 需要压缩
        CC->>AC: call_llm(task="compression")
        AC->>CP: 选择凭证
        AC-->>CC: 摘要结果
        CC-->>AIAgent: 压缩后的消息列表
    end
    
    AIAgent->>AIAgent: 调用 LLM API
    
    alt API 调用失败
        AIAgent->>EC: classify_api_error(error)
        EC-->>AIAgent: ClassifiedError{reason, should_compress, retryable...}
        Note over AIAgent: 根据分类结果：重试/轮换/压缩/中止
    end
    
    AIAgent-->>User: 返回响应
    AIAgent->>MM: sync_all(user, assistant)
```

**核心设计原则：**

1. **无状态函数优先：** `prompt_builder`、`retry_utils`、`prompt_caching`、`redact` 均为纯函数模块，无副作用，易于测试。

2. **策略-机制分离：** `ContextEngine` 定义接口（策略），`ContextCompressor` 提供实现（机制）；`MemoryProvider` 定义接口，插件提供实现。两者均可通过配置替换。

3. **容错隔离：** `MemoryManager` 的每个 Provider 调用都有独立 try-except，一个 Provider 的失败不影响另一个。`auxiliary_client` 的提供者检测链在任一环节失败时透明降级。

4. **分层缓存：** 技能索引用进程内 LRU + 磁盘快照；模型元数据用内存缓存（1h TTL）+ 端点缓存（5m TTL）+ 持久 YAML 缓存。

5. **安全纵深：** 上下文文件扫描（提示注入防护）→ 引用路径沙箱 → 敏感路径阻止 → 输出脱敏 → 环境变量快照锁定，形成多层防御。
