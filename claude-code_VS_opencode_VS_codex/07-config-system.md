# 7. 配置系统

## 7.1 基本对比

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **配置格式** | JSON | JSONC (支持注释) | TOML |
| **配置文件** | `~/.claude.json` + `~/.claude/settings.json` | `opencode.jsonc` (多位置) | `~/.codex/config.toml` |
| **层数** | 5 层 | ~9 层 | 6 层 |
| **指令文件** | `CLAUDE.md`, `CLAUDE.local.md`, `.claude/rules/*.md` | `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md` | `AGENTS.md`, `AGENTS.override.md` |
| **变量替换** | 无 (仅 `@include` 指令) | `{env:VAR}`, `{file:path}` | 无 |
| **企业管控** | managed-settings.json + drop-in 目录 | managed dir + macOS MDM plist | managed_config.toml + macOS MDM (base64 TOML) |

## 7.2 配置合并顺序

### Claude Code (低 → 高优先级)
1. `userSettings` — `~/.claude/settings.json`
2. `projectSettings` — 共享的项目级配置
3. `localSettings` — gitignored 本地覆盖
4. `flagSettings` — `--settings` CLI 参数
5. `policySettings` — managed/enterprise 设置

### OpenCode (低 → 高优先级)
1. Well-known 远程配置 (`/.well-known/opencode`)
2. 全局配置 (`$XDG_CONFIG_HOME/opencode/`)
3. 自定义配置 (`OPENCODE_CONFIG` 标志)
4. 项目配置文件 (从 cwd 向上查找)
5. `.opencode/` 目录配置 (agents/commands/modes/plugins)
6. `OPENCODE_CONFIG_CONTENT` 环境变量
7. Console/org 远程配置
8. Managed 配置目录 (`/etc/opencode` 等)
9. macOS managed preferences (MDM `.mobileconfig`)

### Codex (低 → 高优先级)
1. `Mdm` — macOS managed preferences
2. `System` — `/etc/codex/config.toml` 或 managed config
3. `User` — `$CODEX_HOME/config.toml`
4. `Project` — `.codex/` 目录 (cwd 到项目根)
5. `SessionFlags` — CLI `-c`/`--config` 覆盖
6. `LegacyManagedConfig` — 旧版 managed config (最高优先级)

## 7.3 指令文件发现

### Claude Code
```
发现路径 (低 → 高优先级):
1. /etc/claude-code/CLAUDE.md (managed)
2. ~/.claude/CLAUDE.md (用户级)
3. CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md (从 cwd 向上遍历到根目录)
4. CLAUDE.local.md (项目根, gitignored)

特性:
- 支持 @include 指令: @path, @./relative, @~/home, @/absolute
- 从目录树向上遍历, 所有层级的 CLAUDE.md 都会被合并
```

### OpenCode
```
发现路径:
1. $XDG_CONFIG_HOME/opencode/AGENTS.md 或 ~/.claude/CLAUDE.md (全局)
2. AGENTS.md | CLAUDE.md | CONTEXT.md (findUp 从 cwd 到 worktree root, 先找到先用)

特性:
- config 中 instructions 数组支持: 文件 glob, 相对路径 (~/), HTTP(S) URL
- {env:VAR} 和 {file:path} 变量替换
```

### Codex
```
发现路径:
1. AGENTS.override.md (优先检查)
2. AGENTS.md (从项目根向下到 cwd, 全部拼接)

特性:
- 可配置 project_root_markers (默认: .git)
- 可配置 project_doc_fallback_filenames
- 每文件最大 32 KiB (静默截断)
```

## 7.4 企业/MDM 支持

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **Managed 路径** | macOS: `/Library/Application Support/ClaudeCode`<br/>Windows: `C:\Program Files\ClaudeCode`<br/>Linux: `/etc/claude-code` | macOS: `/Library/Application Support/opencode`<br/>Windows: `C:\ProgramData\opencode`<br/>Linux: `/etc/opencode` | Unix: `/etc/codex/`<br/>Windows: `$CODEX_HOME/managed_config.toml` |
| **Drop-in** | `managed-settings.d/*.json` (systemd 风格, 字母序合并) | 无 | 无 |
| **MDM** | 通过 managed settings 文件路径 | macOS plist domain `ai.opencode.managed` | macOS base64 编码 TOML |
| **远程策略** | 通过 `policySettings` | Console/org API | Cloud requirements 系统 |

## 7.5 评价

- **Claude Code**: 层级最少 (5层) 但最清晰，`@include` 指令强大，managed drop-in 目录类似 systemd
- **OpenCode**: 层级最多 (9层)，支持远程配置 + HTTP URL 指令 + 变量替换 — **最灵活**
- **Codex**: TOML 格式可读性好，AGENTS.override.md 提供简洁的本地覆盖，文件大小限制 (32KB) 防止意外加载巨型文件
