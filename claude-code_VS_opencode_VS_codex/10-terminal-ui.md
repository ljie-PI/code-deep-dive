# 10. 终端 UI

## 10.1 框架对比

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **框架** | React 19 + Ink (自研 fork) | SolidJS + @opentui/solid | ratatui + crossterm |
| **渲染** | React reconciler → Yoga (flexbox) → ANSI | SolidJS reactive → opentui → ANSI | 即时模式 widget → ANSI |
| **组件数** | ~351 .tsx 文件 | ~74 .tsx 文件 | ~164 .rs 文件 |
| **核心组件** | `REPL.tsx` (6,182 行) | `app.tsx` + `session/index.tsx` (2,288 行) | `app.rs` + `chatwidget.rs` |
| **线程模型** | 单线程 (UI + Agent 同进程) | Worker 线程 (TUI 在独立 worker 中) | tokio 异步 (TUI 事件循环单线程, I/O 异步) |
| **Markdown** | `marked` 库 + token 缓存 (max 500) + `cliHighlight` | 未详细调查 | `pulldown_cmark` → ratatui Span/Text + `highlight_code_to_lines` |
| **主题** | 2 种 (light/dark, 自动检测) | 33 种命名主题 (catppuccin/dracula/gruvbox/nord/tokyonight 等) | 自适应 (检测终端背景色, alpha 混合) |
| **Vim 支持** | 有 (专用 `src/vim/` 模块) | 无 (keybind 系统) | 有 (vim-like 导航) |
| **输入** | VimTextInput + 粘贴 + 剪贴板图片 + 语音 | KeybindProvider + 命令面板 + 历史/frecency/stash | crossterm 事件 + 外部编辑器 + 剪贴板 + 语音 |

## 10.2 渲染方式

### Claude Code: React Reconciler
```
React Component Tree
  ↓ (React 19 reconciler)
Yoga Layout (flexbox)
  ↓
ANSI escape codes → Terminal
```
- 自研 `src/ink/reconciler.ts` (512 行) — 完全控制渲染管线
- Ink 是 React 的终端渲染器，Claude Code fork 了它以解决 Ink 原版的限制
- `REPL.tsx` 6,182 行是一个巨型组件 — UI 和业务逻辑高度耦合

### OpenCode: SolidJS Reactive
```
SolidJS Signals/Effects
  ↓ (reactive updates)
@opentui/core Renderer
  ↓
ANSI escape codes → Terminal
```
- 不是 React — SolidJS 使用信号/效果做细粒度响应式更新
- TUI 在独立 worker 线程中运行，与 server 分离
- 路由系统: `Home` 和 `Session` 视图

### Codex: Immediate Mode (ratatui)
```
Application State
  ↓ (每帧完整重绘)
ratatui Widget 组合
  ↓ (crossterm backend)
ANSI escape codes → Terminal
```
- 即时模式: 每个 tick 完整重绘整个界面
- Widget 组合: 声明式但非响应式
- 自定义 Markdown 渲染: `pulldown_cmark` 解析 → ratatui `Text`/`Span` → 自适应换行

## 10.3 主题系统

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **主题数量** | 2 (light/dark) | 33 (JSON 文件) | 0 (自适应) |
| **检测** | `getSystemThemeName()` 自动检测 | 用户选择 | `terminal_palette.rs` 检测背景色 |
| **适配** | ThemeProvider React 上下文 | JSON 主题文件加载 | Alpha 混合算法自适应 |
| **定制** | 简单 (二选一) | 最丰富 (33种 + 自定义) | 最智能 (自动适应任何终端) |

## 10.4 评价

| 维度 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **UI 复杂度** | 最高 (351 组件, 6K 行 REPL) | 中 (74 组件) | 中 (164 文件) |
| **UI/Engine 耦合** | 紧密 (同进程) | 松散 (Worker 分离) | 松散 (异步任务) |
| **主题** | 最简单 | 最丰富 | 最智能 |
| **性能** | 受限于 React reconciler 开销 | SolidJS 细粒度更新 | 即时模式 + Rust 性能 |
| **可维护性** | 低 (巨型 REPL.tsx) | 高 (小组件 + 清晰分离) | 中 (Rust 类型系统保障) |
