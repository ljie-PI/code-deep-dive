# 附录 C：环境变量速查

> 所有环境变量定义在 `packages/opencode/src/flag/flag.ts`。

## 核心配置

| 环境变量 | 类型 | 默认值 | 描述 |
|---------|------|--------|------|
| `OPENCODE_CONFIG` | string | — | 自定义配置文件路径 |
| `OPENCODE_CONFIG_DIR` | string | — | 自定义配置目录路径 |
| `OPENCODE_CONFIG_CONTENT` | string | — | 内联配置内容（JSON） |
| `OPENCODE_DB` | string | — | 自定义数据库路径 |
| `OPENCODE_SERVER_PASSWORD` | string | — | HTTP Server 认证密码 |
| `OPENCODE_SERVER_USERNAME` | string | — | HTTP Server 认证用户名 |
| `OPENCODE_PERMISSION` | string | — | 全局权限模式 |
| `OPENCODE_CLIENT` | string | `"cli"` | 客户端标识 |
| `OPENCODE_GIT_BASH_PATH` | string | — | Windows 上 Git Bash 路径 |

## 功能开关 (布尔值: "true"/"1" 启用)

| 环境变量 | 默认 | 描述 |
|---------|------|------|
| `OPENCODE_DISABLE_AUTOUPDATE` | false | 禁用自动更新检查 |
| `OPENCODE_DISABLE_AUTOCOMPACT` | false | 禁用自动上下文压缩 |
| `OPENCODE_DISABLE_PRUNE` | false | 禁用快照清理 |
| `OPENCODE_DISABLE_TERMINAL_TITLE` | false | 禁用终端标题修改 |
| `OPENCODE_DISABLE_DEFAULT_PLUGINS` | false | 禁用默认插件 |
| `OPENCODE_DISABLE_LSP_DOWNLOAD` | false | 禁用 LSP 服务器自动下载 |
| `OPENCODE_DISABLE_MODELS_FETCH` | false | 禁用远程模型列表获取 |
| `OPENCODE_DISABLE_MOUSE` | false | 禁用 TUI 鼠标支持 |
| `OPENCODE_DISABLE_PROJECT_CONFIG` | false | 禁用项目级配置 |
| `OPENCODE_DISABLE_EMBEDDED_WEB_UI` | false | 禁用嵌入式 Web UI |
| `OPENCODE_DISABLE_CHANNEL_DB` | false | 禁用频道数据库 |
| `OPENCODE_SKIP_MIGRATIONS` | false | 跳过数据库迁移 |
| `OPENCODE_STRICT_CONFIG_DEPS` | false | 严格检查配置依赖 |
| `OPENCODE_ALWAYS_NOTIFY_UPDATE` | false | 总是通知更新 |
| `OPENCODE_AUTO_SHARE` | false | 自动分享会话 |
| `OPENCODE_AUTO_HEAP_SNAPSHOT` | false | 自动堆快照 |
| `OPENCODE_SHOW_TTFD` | false | 显示首次展示延迟 |

## Claude Code 兼容性

| 环境变量 | 默认 | 描述 |
|---------|------|------|
| `OPENCODE_DISABLE_CLAUDE_CODE` | false | 禁用所有 Claude Code 兼容 |
| `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT` | false | 禁用 Claude Code 提示 |
| `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS` | false | 禁用 Claude Code Skills |
| `OPENCODE_DISABLE_EXTERNAL_SKILLS` | false | 禁用外部 Skills |

## 实验性功能

| 环境变量 | 默认 | 描述 |
|---------|------|------|
| `OPENCODE_EXPERIMENTAL` | false | 启用所有实验性功能 |
| `OPENCODE_EXPERIMENTAL_FILEWATCHER` | false | 实验性文件监视 |
| `OPENCODE_EXPERIMENTAL_DISABLE_FILEWATCHER` | false | 禁用实验性文件监视 |
| `OPENCODE_EXPERIMENTAL_ICON_DISCOVERY` | false | 实验性图标发现 |
| `OPENCODE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT` | Win32 | 禁用选中复制 |
| `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS` | — | Bash 工具默认超时 (ms) |
| `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX` | — | 最大输出令牌数 |
| `OPENCODE_EXPERIMENTAL_OXFMT` | false | 实验性代码格式化 |
| `OPENCODE_EXPERIMENTAL_LSP_TY` | false | 实验性 LSP 类型 |
| `OPENCODE_EXPERIMENTAL_LSP_TOOL` | false | 实验性 LSP 工具 |
| `OPENCODE_EXPERIMENTAL_PLAN_MODE` | false | 实验性计划模式 |
| `OPENCODE_EXPERIMENTAL_WORKSPACES` | false | 实验性多工作区 |
| `OPENCODE_EXPERIMENTAL_MARKDOWN` | true | 实验性 Markdown 渲染 |
| `OPENCODE_ENABLE_EXPERIMENTAL_MODELS` | false | 启用实验性模型 |
| `OPENCODE_ENABLE_QUESTION_TOOL` | false | 启用提问工具 |
| `OPENCODE_ENABLE_EXA` | false | 启用 Exa 搜索 |
| `OPENCODE_DISABLE_FILETIME_CHECK` | false | 禁用文件时间检查 |

## 模型配置

| 环境变量 | 类型 | 描述 |
|---------|------|------|
| `OPENCODE_MODELS_URL` | string | 自定义模型列表 URL |
| `OPENCODE_MODELS_PATH` | string | 自定义模型列表文件路径 |

## OpenTelemetry

| 环境变量 | 类型 | 描述 |
|---------|------|------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | string | OTLP 导出端点 |
| `OTEL_EXPORTER_OTLP_HEADERS` | string | OTLP 导出头 |

## 测试/调试

| 环境变量 | 类型 | 描述 |
|---------|------|------|
| `OPENCODE_PURE` | boolean | 纯模式（禁用副作用） |
| `OPENCODE_FAKE_VCS` | string | 模拟 VCS 类型 |
| `OPENCODE_TUI_CONFIG` | string | TUI 配置覆盖 |
| `OPENCODE_PLUGIN_META_FILE` | string | 插件元数据文件路径 |
| `OPENCODE_TEST_MANAGED_CONFIG_DIR` | string | 测试用托管配置目录 |
