# 附录

---

## A. 关键文件索引

### 顶层文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `run_agent.py` | 10,524 | AIAgent 类 — 核心对话循环 |
| `cli.py` | 9,878 | HermesCLI — 交互式 TUI |
| `model_tools.py` | 577 | 工具编排公共 API |
| `toolsets.py` | 655 | 工具集定义与组合 |
| `toolset_distributions.py` | 364 | 工具分布逻辑 |
| `hermes_constants.py` | 269 | 共享常量（路径、URL） |
| `hermes_state.py` | 1,238 | SQLite 状态存储（会话、消息、FTS5） |
| `hermes_logging.py` | 394 | 日志配置 |
| `hermes_time.py` | 104 | 时间工具 |
| `utils.py` | 214 | 通用工具函数 |
| `batch_runner.py` | 1,287 | 批量轨迹生成 |
| `trajectory_compressor.py` | 1,457 | 轨迹压缩 |
| `mcp_serve.py` | 867 | MCP 服务器模式 |
| `rl_cli.py` | 446 | RL 训练 CLI |

### agent/ 包

| 文件 | 职责 |
|------|------|
| `prompt_builder.py` | 系统提示词构建 |
| `context_compressor.py` | 上下文压缩 |
| `memory_manager.py` | 记忆加载与格式化 |
| `memory_provider.py` | 记忆存储后端 |
| `model_metadata.py` | 模型能力检测与 token 估算 |
| `smart_model_routing.py` | 动态模型路由 |
| `auxiliary_client.py` | 辅助 LLM 客户端 |
| `anthropic_adapter.py` | Anthropic API 适配 |
| `context_engine.py` | 上下文引用管理 |
| `context_references.py` | 文件/URL 上下文引用 |
| `prompt_caching.py` | Anthropic 提示词缓存 |
| `error_classifier.py` | API 错误分类 |
| `retry_utils.py` | 重试与退避策略 |
| `credential_pool.py` | API 密钥轮换 |
| `rate_limit_tracker.py` | 速率限制追踪 |
| `trajectory.py` | 轨迹序列化 |
| `usage_pricing.py` | 使用计费 |
| `display.py` | 终端显示辅助 |
| `insights.py` | 使用分析 |
| `title_generator.py` | 会话标题自动生成 |
| `skill_commands.py` | 技能 slash 命令 |
| `skill_utils.py` | 技能工具函数 |
| `subdirectory_hints.py` | 工作目录提示追踪 |
| `redact.py` | 敏感数据脱敏 |
| `copilot_acp_client.py` | ACP 客户端 |

### tools/ 目录 — 已注册工具清单

| 工具名 | 文件 | 工具集 | 用途 |
|--------|------|--------|------|
| `read_file` | file_tools.py:832 | file | 读取文件内容 |
| `write_file` | file_tools.py:833 | file | 写入文件 |
| `patch` | file_tools.py:834 | file | 增量编辑文件 |
| `search_files` | file_tools.py:835 | file | 搜索文件内容 |
| `terminal` | terminal_tool.py:1769 | terminal | 执行终端命令 |
| `process` | process_registry.py:1166 | terminal | 管理后台进程 |
| `web_search` | web_tools.py:2082 | web | Web 搜索 |
| `web_extract` | web_tools.py:2092 | web | 提取网页内容 |
| `browser_navigate` | browser_tool.py:2306 | browser | 导航到 URL |
| `browser_snapshot` | browser_tool.py:2314 | browser | 截取页面快照 |
| `browser_click` | browser_tool.py:2323 | browser | 点击页面元素 |
| `browser_type` | browser_tool.py:2331 | browser | 输入文本 |
| `browser_scroll` | browser_tool.py:2339 | browser | 滚动页面 |
| `browser_back` | browser_tool.py:2347 | browser | 后退导航 |
| `browser_press` | browser_tool.py:2355 | browser | 按键操作 |
| `browser_get_images` | browser_tool.py:2364 | browser | 获取页面图片 |
| `browser_vision` | browser_tool.py:2372 | browser | 视觉分析页面 |
| `browser_console` | browser_tool.py:2380 | browser | 浏览器控制台 |
| `vision_analyze` | vision_tools.py:790 | vision | 图像分析 |
| `image_generate` | image_generation_tool.py:694 | image | 图像生成 |
| `text_to_speech` | tts_tool.py:1050 | tts | 文本转语音 |
| `memory` | memory_tool.py:544 | memory | 持久化记忆操作 |
| `todo` | todo_tool.py:269 | todo | 任务管理 |
| `session_search` | session_search_tool.py:492 | session | 搜索历史会话 |
| `clarify` | clarify_tool.py:131 | clarify | 向用户提问 |
| `execute_code` | code_execution_tool.py:1367 | code | 编程式工具调用 |
| `delegate_task` | delegate_tool.py:1088 | delegate | 子代理委托 |
| `mixture_of_agents` | mixture_of_agents_tool.py:553 | delegate | 混合代理模式 |
| `cronjob` | cronjob_tools.py:518 | cron | 定时任务管理 |
| `send_message` | send_message_tool.py:1042 | messaging | 跨平台消息发送 |
| `skills_list` | skills_tool.py:1339 | skills | 列出可用技能 |
| `skill_view` | skills_tool.py:1349 | skills | 查看技能内容 |
| `skill_manage` | skill_manager_tool.py:746 | skills | 技能 CRUD |
| `ha_list_entities` | homeassistant_tool.py:456 | homeassistant | 列出 HA 实体 |
| `ha_get_state` | homeassistant_tool.py:465 | homeassistant | 获取 HA 状态 |
| `ha_list_services` | homeassistant_tool.py:474 | homeassistant | 列出 HA 服务 |
| `ha_call_service` | homeassistant_tool.py:483 | homeassistant | 调用 HA 服务 |
| `rl_list_environments` | rl_training_tool.py:1376 | rl | 列出 RL 环境 |
| `rl_select_environment` | rl_training_tool.py:1378 | rl | 选择 RL 环境 |
| `rl_get_current_config` | rl_training_tool.py:1380 | rl | 获取 RL 配置 |
| `rl_edit_config` | rl_training_tool.py:1382 | rl | 编辑 RL 配置 |
| `rl_start_training` | rl_training_tool.py:1384 | rl | 启动 RL 训练 |
| `rl_check_status` | rl_training_tool.py:1386 | rl | 检查训练状态 |
| `rl_stop_training` | rl_training_tool.py:1388 | rl | 停止 RL 训练 |
| `rl_get_results` | rl_training_tool.py:1390 | rl | 获取训练结果 |
| `rl_list_runs` | rl_training_tool.py:1392 | rl | 列出训练运行 |
| `rl_test_inference` | rl_training_tool.py:1394 | rl | 测试推理 |

*注：MCP 工具通过 `mcp_tool.py` 在运行时动态注册，不在上表中。*

### gateway/platforms/ — 消息平台

| 文件 | 平台 | 行数 |
|------|------|------|
| `base.py` | 基类 | 1,998 |
| `telegram.py` | Telegram | 2,780 |
| `discord.py` | Discord | 2,957 |
| `feishu.py` | 飞书 (Lark) | 3,623 |
| `matrix.py` | Matrix | 2,045 |
| `api_server.py` | REST API | 1,838 |
| `slack.py` | Slack | — |
| `whatsapp.py` | WhatsApp | — |
| `signal.py` | Signal | — |
| `email.py` | Email | — |
| `homeassistant.py` | Home Assistant | — |
| `dingtalk.py` | 钉钉 | — |
| `webhook.py` | Webhook | — |
| `bluebubbles.py` | BlueBubbles (iMessage) | — |
| `sms.py` | SMS | — |
| `wecom.py` | 企业微信 | — |
| `weixin.py` | 微信公众号 | — |
| `mattermost.py` | Mattermost | — |

### tools/environments/ — 终端后端

| 文件 | 后端 | 特点 |
|------|------|------|
| `local.py` | 本地 | 直接执行 |
| `docker.py` | Docker | 容器隔离 |
| `ssh.py` | SSH | 远程执行 |
| `daytona.py` | Daytona | 无服务器持久化 |
| `modal.py` | Modal | 无服务器 GPU |
| `singularity.py` | Singularity | HPC 容器 |

---

## B. 术语表

| 英文 | 中文 | 含义 |
|------|------|------|
| Agent | 代理 | AI 代理实例，能自主执行任务 |
| AIAgent | AI 代理 | `run_agent.py` 中的核心类 |
| Tool | 工具 | 代理可调用的原子操作（Python 实现） |
| Toolset | 工具集 | 工具的逻辑分组 |
| Skill | 技能 | Markdown 格式的可复用指令集 |
| Skills Hub | 技能中心 | 社区技能发现与分享平台 |
| Gateway | 网关 | 多平台消息路由进程 |
| Platform | 平台 | 消息平台（Telegram、Discord 等） |
| Session | 会话 | 一次对话的完整上下文 |
| Context | 上下文 | LLM 的输入窗口内容 |
| Compression | 压缩 | 对话历史的摘要/裁剪 |
| Memory | 记忆 | 跨会话持久化的信息 |
| Nudge | 提示 | 系统提示词中鼓励代理保存记忆的指令 |
| SOUL.md | 灵魂文件 | 定义代理人格的 Markdown 文件 |
| Context File | 上下文文件 | 项目级上下文信息文件 |
| HERMES_HOME | 主目录 | 代理数据目录，默认 `~/.hermes/` |
| Trajectory | 轨迹 | 完整对话记录（用于训练） |
| Iteration Budget | 迭代预算 | 单次对话的最大工具调用次数 |
| Failover | 故障转移 | API 错误时自动切换 provider |
| Credential Pool | 密钥池 | 多个 API 密钥的轮换管理 |
| Auxiliary Client | 辅助客户端 | 用于非对话任务的独立 LLM 客户端 |
| CamoFox | CamoFox | 基于 Firefox 的反检测浏览器引擎 |
| MCP | Model Context Protocol | 模型上下文协议 — 工具扩展标准 |
| ACP | Agent Communication Protocol | 代理通信协议 — 编辑器集成标准 |
| FTS5 | Full-Text Search 5 | SQLite 全文搜索扩展 |
| WAL | Write-Ahead Logging | SQLite 并发写入模式 |
| Honcho | Honcho | 辩证式用户建模服务 |
| Atropos | Atropos | Nous Research 的 RL 训练框架 |
| OpenRouter | OpenRouter | 多模型聚合 API 服务 |

---

## C. 环境变量速查（部分）

系统使用约 305 个环境变量，以下列出最重要的：

### 核心配置

| 变量 | 含义 | 默认值 |
|------|------|--------|
| `HERMES_HOME` | 代理主目录 | `~/.hermes` |
| `HERMES_QUIET` | 抑制启动消息 | 未设置 |
| `HERMES_OPTIONAL_SKILLS` | 可选技能目录 | `$HERMES_HOME/optional-skills` |

### Provider API 密钥

| 变量 | Provider |
|------|----------|
| `OPENROUTER_API_KEY` | OpenRouter |
| `OPENAI_API_KEY` | OpenAI |
| `ANTHROPIC_API_KEY` | Anthropic |
| `AI_GATEWAY_API_KEY` | AI Gateway |
| `CUSTOM_API_KEY` | 自定义端点 |
| `ELEVENLABS_API_KEY` | ElevenLabs TTS |

### 消息平台

| 变量 | 平台 |
|------|------|
| `DISCORD_BOT_TOKEN` | Discord |
| `TELEGRAM_BOT_TOKEN` | Telegram (via config) |
| `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` | Slack |
| `DINGTALK_CLIENT_ID` / `DINGTALK_CLIENT_SECRET` | 钉钉 |
| `EMAIL_ADDRESS` | Email |
| `HASS_TOKEN` / `HASS_URL` | Home Assistant |

### 终端后端

| 变量 | 后端 |
|------|------|
| `DAYTONA_API_KEY` | Daytona |
| `MODAL_TOKEN_ID` / `MODAL_TOKEN_SECRET` | Modal |
| `CAMOFOX_URL` | CamoFox 浏览器 |

### 代理行为

| 变量 | 含义 |
|------|------|
| `API_TIMEOUT_SECONDS` | API 请求超时 |
| `API_BASE_URL` | 自定义 API 端点 |
| `DELEGATION_MAX_CONCURRENT_CHILDREN` | 最大并发子代理 |
| `AUXILIARY_VISION_MODEL` | 辅助视觉模型 |
| `AUXILIARY_WEB_EXTRACT_MODEL` | 辅助网页提取模型 |

---

## D. 工具集定义速查

来源：`toolsets.py`

| 工具集 | 包含的工具 |
|--------|-----------|
| `web` | web_search, web_extract |
| `search` | web_search |
| `terminal` | terminal, process |
| `file` | read_file, write_file, patch, search_files |
| `browser` | browser_navigate, browser_snapshot, browser_click, browser_type, browser_scroll, browser_back, browser_press, browser_get_images, browser_vision, browser_console |
| `vision` | vision_analyze |
| `image` | image_generate |
| `tts` | text_to_speech |
| `memory` | memory |
| `todo` | todo |
| `session` | session_search |
| `skills` | skills_list, skill_view, skill_manage |
| `delegate` | execute_code, delegate_task |
| `cron` | cronjob |
| `messaging` | send_message |
| `homeassistant` | ha_list_entities, ha_get_state, ha_list_services, ha_call_service |
| `rl` | rl_list_environments, rl_select_environment, ... (10 个工具) |

`_HERMES_CORE_TOOLS` 列表（`toolsets.py:31-63`）定义了 CLI 和所有消息平台共享的标准工具集。
