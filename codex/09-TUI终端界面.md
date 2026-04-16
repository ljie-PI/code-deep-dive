# 第九章 TUI 终端界面

## 概述

Codex 的 TUI (Terminal User Interface) 是用户与 AI 代理交互的主界面。它不是一个简单的命令行 REPL,而是一个功能丰富的终端应用程序,基于 Ratatui 框架构建,支持富文本渲染、Markdown 展示、差异对比、文件搜索、流式输出等特性。

TUI 的设计采用了分层的事件驱动架构:底层的终端处理(按键、绘制)与上层的业务逻辑(对话管理、审批流程)通过事件系统解耦。本章将从应用启动流程开始,深入分析各个 UI 组件的职责和交互方式。

## 启动流程

### 从 main 到 App

Codex TUI 的启动是一个多层初始化过程:

```
main.rs
  └─> lib.rs::run_main()
       ├─> ConfigBuilder::build()        // 加载配置
       ├─> AppServerClient::connect()    // 连接 app-server
       ├─> AppServerSession::new()       // 创建会话
       ├─> tui::init()                   // 初始化终端
       │    ├─> set_modes()              // 设置终端模式
       │    ├─> flush_terminal_input_buffer() // 清空输入缓冲
       │    ├─> set_panic_hook()         // 安装 panic 恢复钩子
       │    └─> CustomTerminal::with_options() // 创建终端后端
       └─> App::run()                    // 启动主循环
```

`run_main()` 是实际的入口函数(定义在 `tui/src/main.rs:665`)。它协调以下初始化步骤:

1. **配置构建**:通过 `ConfigBuilder` 加载并合并多层配置(全局、项目、命令行参数)
2. **服务连接**:创建 `AppServerClient` 连接到 app-server(可以是进程内或远程)
3. **会话创建**:创建 `AppServerSession`,管理与 app-server 的通信
4. **终端初始化**:调用 `tui::init()` 设置终端模式
5. **应用启动**:创建 `App` 实例并进入主事件循环

### tui::init():终端初始化

```rust
pub fn init() -> Result<Terminal> {
    if !stdin().is_terminal() { return Err(...); }
    if !stdout().is_terminal() { return Err(...); }
    set_modes()?;
    flush_terminal_input_buffer();
    set_panic_hook();
    let backend = CrosstermBackend::new(stdout());
    let tui = CustomTerminal::with_options(backend)?;
    Ok(tui)
}
```

初始化首先验证 stdin 和 stdout 确实是终端(非管道),然后进行终端模式设置。

## Tui 结构体:终端管理层

### 定义

`Tui` 结构体(定义在 `tui/src/tui.rs:280`)是终端底层管理器,封装了所有与终端交互的细节:

```rust
pub struct Tui {
    frame_requester: FrameRequester,      // 帧请求器(控制重绘节奏)
    draw_tx: broadcast::Sender<()>,       // 绘制信号发送端
    event_broker: Arc<EventBroker>,       // 事件分发器
    pub(crate) terminal: Terminal,        // Ratatui 终端实例
    pending_history_lines: Vec<Line<'static>>, // 待写入滚动缓冲的行
    alt_saved_viewport: Option<Rect>,     // 备用屏幕保存的视口
    alt_screen_active: Arc<AtomicBool>,   // 是否处于备用屏幕
    terminal_focused: Arc<AtomicBool>,    // 终端是否获得焦点
    enhanced_keys_supported: bool,        // 是否支持增强键盘
    notification_backend: Option<DesktopNotificationBackend>, // 桌面通知后端
    notification_condition: NotificationCondition, // 通知触发条件
    is_zellij: bool,                      // 是否运行在 Zellij 中
    alt_screen_enabled: bool,             // 是否启用备用屏幕
}
```

### 终端模式设置

`set_modes()` 函数配置 crossterm 的终端模式:

```rust
pub fn set_modes() -> Result<()> {
    execute!(stdout(), EnableBracketedPaste)?;  // 启用括号粘贴模式
    enable_raw_mode()?;                          // 启用原始模式
    // 启用键盘增强标志(修饰键识别)
    let _ = execute!(stdout(), PushKeyboardEnhancementFlags(
        KeyboardEnhancementFlags::DISAMBIGUATE_ESCAPE_CODES
            | KeyboardEnhancementFlags::REPORT_EVENT_TYPES
            | KeyboardEnhancementFlags::REPORT_ALTERNATE_KEYS
    ));
    let _ = execute!(stdout(), EnableFocusChange); // 启用焦点变化事件
    Ok(())
}
```

四种终端模式各有用途:

**括号粘贴模式(Bracketed Paste)**:让终端将粘贴的文本用特殊转义序列包裹,使应用能区分粘贴和逐键输入。这对 chat composer 很重要——粘贴的多行文本应作为一次操作处理。

**原始模式(Raw Mode)**:禁用终端的行缓冲和信号处理,让应用直接接收每个按键。这是所有 TUI 应用的基础。

**键盘增强(Keyboard Enhancement)**:crossterm 的扩展功能,支持区分 Enter 和 Shift+Enter 等修饰键组合。这在 chat composer 中用于区分"提交"和"换行"。并非所有终端都支持——代码优雅降级:

```rust
let _ = execute!(stdout(), PushKeyboardEnhancementFlags(...));
```

使用 `let _ =` 忽略错误,确保在不支持的终端上不会崩溃。

**焦点变化(Focus Change)**:让应用感知终端窗口的焦点状态。Codex 使用它来决定是否发送桌面通知(仅在窗口失焦时通知)。

### 恢复与 Panic 处理

`restore()` 函数是 `set_modes()` 的逆操作:

```rust
pub fn restore() -> Result<()> {
    let _ = execute!(stdout(), PopKeyboardEnhancementFlags);
    execute!(stdout(), DisableBracketedPaste)?;
    let _ = execute!(stdout(), DisableFocusChange);
    disable_raw_mode()?;
    let _ = execute!(stdout(), crossterm::cursor::Show);
    Ok(())
}
```

`set_panic_hook()` 安装自定义的 panic 处理器,确保即使应用崩溃也能恢复终端状态:

```rust
fn set_panic_hook() {
    let hook = panic::take_hook();
    panic::set_hook(Box::new(move |panic_info| {
        let _ = restore();  // 先恢复终端
        hook(panic_info);   // 再执行原来的 panic 处理
    }));
}
```

这是 TUI 应用的关键安全措施。如果 panic 时不恢复终端,用户的 shell 会处于原始模式,导致终端不可用。

### 事件系统

`Tui` 使用 `EventBroker` 管理事件分发。`TuiEvent` 枚举定义了三种终端事件:

```rust
pub enum TuiEvent {
    Key(KeyEvent),    // 按键事件
    Paste(String),    // 粘贴事件(括号粘贴模式)
    Draw,             // 绘制请求
}
```

`FrameRequester` 控制重绘频率。`TARGET_FRAME_INTERVAL` 定义了最小帧间隔,防止过于频繁的重绘浪费 CPU。

### 平台特定:输入缓冲清空

Codex 在初始化时清空终端输入缓冲,防止残留的按键干扰:

**Unix**:通过 `tcflush(STDIN_FILENO, TCIFLUSH)` 清空 stdin 队列。

**Windows**:通过 `FlushConsoleInputBuffer` Win32 API 清空控制台输入缓冲。

### Zellij 兼容性

检测是否运行在 Zellij 终端复用器中。Zellij 的滚动行为与标准终端不同,需要禁用备用屏幕(alternate screen)来保持滚动缓冲正常工作。

## App 结构体:核心状态持有者

### 职责

`App`(定义在 `tui/src/app.rs`)是 TUI 的核心状态管理器。从它的 import 列表可以看出其职责范围极广:

- 管理 `ChatWidget`(聊天界面)和 `AppServerSession`(服务通信)
- 持有 `Config`(配置)和 `ModelCatalog`(模型目录)
- 处理 `AppEvent`(应用事件)和 `AppCommand`(应用命令)
- 协调审批流程、文件搜索、插件管理等

### 事件路由

键盘输入经过多层路由到达最终的处理器:

```
按键输入
  └─> Tui (事件采集)
       └─> App (全局快捷键处理)
            └─> ChatWidget (聊天级事件处理)
                 └─> BottomPane (底部面板路由)
                      ├─> BottomPaneView (模态弹窗,如果可见)
                      └─> ChatComposer (文本编辑器)
```

每一层都可以"消费"事件(返回已处理),阻止事件继续向下传播。这种分层设计允许:

- `App` 层截获全局快捷键(如退出、中断)
- `ChatWidget` 层处理聊天相关快捷键(如 Ctrl+T 打开记录覆盖层)
- `BottomPane` 层决定事件去向(模态弹窗还是编辑器)
- `ChatComposer` 层处理文本编辑操作

### AppEvent 与 AppCommand

`AppEvent` 定义了所有可能的应用事件类型:

- 退出模式(`ExitMode`)
- 反馈分类(`FeedbackCategory`)
- 音频设备选择(`RealtimeAudioDeviceKind`)
- 速率限制刷新(`RateLimitRefreshOrigin`)
- Windows 沙箱启用(`WindowsSandboxEnableMode`)

`AppCommand` 和 `AppCommandView` 定义了可执行的应用命令,对应斜杠命令和快捷键操作。

## ChatWidget:聊天界面

### 架构

`ChatWidget`(定义在 `tui/src/chatwidget.rs`)是聊天界面的主组件。它的文档注释详细描述了其设计:

> UI has both committed transcript cells (finalized HistoryCells) and an in-flight active cell that can mutate in place while streaming.

两类单元格的设计:

**已提交的单元格(committed cells)**:已完成的对话条目,存储为 `HistoryCell` 列表。每个 `HistoryCell` 代表一轮完整的交互(用户输入 + 助手响应 + 工具调用结果等)。

**活跃单元格(active_cell)**:正在流式传输的当前条目。它可以就地变更(mutate in place),通常代表一组合并的执行/工具调用。流式完成后提升为已提交单元格。

### 记录覆盖层(Transcript Overlay)

Ctrl+T 触发记录覆盖层,以全屏模式显示完整的对话记录。覆盖层的同步通过 `App::overlay_forward_event` 实现,利用 `active_cell_transcript_key()` 和 `active_cell_transcript_lines()` 将活跃单元格的实时尾部缓存并在绘制时刷新。

缓存键的设计让覆盖层能在活跃单元格变化时刷新缓存尾部,同时避免每帧都重建。

### MCP 启动与任务状态

ChatWidget 追踪两种独立的"忙碌"状态:

- `agent_turn_running`:AI 代理正在执行一轮对话
- `mcp_startup_status`:MCP 服务器正在启动

这两种状态通过 `update_task_running_state` 同步到底部面板的单一指示器(spinner 和中断提示)。

### 前序输出(Preamble)

对于支持前序的模型,助手输出可能在最终答案之前包含注释。在流式传输期间,状态行被隐藏以避免重复的进度指示。一旦注释完成且流队列排空,状态行重新显示。

## BottomPane:底部面板

### 结构

`BottomPane`(定义在 `tui/src/bottom_pane/mod.rs`)是聊天 UI 的交互底栏,拥有两个核心子组件:

**ChatComposer**:可编辑的提示输入区域,是用户输入文本的主界面。

**view_stack**:模态弹窗栈。当需要显示选择列表、审批覆盖层等临时 UI 时,弹窗被推入栈中,临时替代 composer 的显示位置。

### 输入路由

底部面板的输入路由遵循优先级规则(源码注释):

> BottomPane decides which local surface receives a key (view vs composer), while higher-level intent such as "interrupt" or "quit" is decided by the parent widget (ChatWidget).

具体的 Ctrl+C 处理流程:
1. 活跃的模态视图优先消费 Ctrl+C(通常用于关闭自身)
2. 活跃的 history search 消费 Ctrl+C(取消搜索)
3. ChatWidget 处理未消费的 Ctrl+C(中断或双击退出)

### 子模块

```
bottom_pane/
  ├── chat_composer.rs           // 文本编辑器
  ├── chat_composer_history.rs   // 输入历史管理
  ├── bottom_pane_view.rs        // 模态视图基础
  ├── approval_overlay.rs        // 命令审批覆盖层
  ├── command_popup.rs           // 斜杠命令弹窗
  ├── file_search_popup.rs       // 文件搜索弹窗
  ├── app_link_view.rs           // 应用链接视图
  ├── mcp_server_elicitation.rs  // MCP 服务器交互
  ├── custom_prompt_view.rs      // 自定义 prompt 视图
  ├── experimental_features_view.rs // 实验功能视图
  ├── feedback_view.rs           // 反馈视图
  ├── footer.rs                  // 页脚渲染
  ├── list_selection_view.rs     // 列表选择视图
  ├── multi_select_picker.rs     // 多选选择器
  ├── pending_input_preview.rs   // 排队输入预览
  ├── pending_thread_approvals.rs // 待审批线程
  ├── status_line_setup.rs       // 状态行配置
  ├── title_setup.rs             // 标题设置
  └── unified_exec_footer.rs     // 统一执行页脚
```

## ChatComposer:文本编辑器

### 核心职责

`ChatComposer`(定义在 `tui/src/bottom_pane/chat_composer.rs`)是 TUI 中最复杂的组件之一。它的文档注释列出了五大职责:

1. **文本编辑**:管理 `TextArea` 输入缓冲区,支持附件占位符("元素")
2. **弹窗路由**:将按键路由到活跃弹窗(斜杠命令、文件搜索、技能/应用提及)
3. **斜杠命令提升**:将输入的斜杠命令转换为原子元素
4. **提交处理**:区分 Enter(提交)和换行,处理 Tab 排队
5. **粘贴处理**:在不支持括号粘贴的平台上(特别是 Windows)模拟粘贴操作

### 按键事件路由

```rust
// 简化的路由逻辑
fn handle_key_event(&mut self, key: KeyEvent) -> HandleResult {
    if self.popup_visible() {
        // 路由到弹窗特定处理器
        return self.handle_key_event_with_popup(key);
    }
    let result = self.handle_key_event_without_popup(key);
    self.sync_popups(); // 每次按键后同步弹窗状态
    result
}
```

### 历史导航

`ChatComposerHistory` 管理输入历史,合并两种来源:

- **持久化跨会话历史**:仅文本,无元素范围或附件
- **本地会话内历史**:完整文本 + 文本元素 + 本地/远程图片附件

方向键 Up/Down 导航历史:
- 回忆本地条目时,恢复文本元素和两种附件(本地图片路径 + 远程图片 URL)
- 回忆持久化条目时,仅恢复文本
- 回忆后光标移至行尾,保持 shell 风格的历史遍历语义

**Ctrl+R**:反向增量搜索。页脚变为搜索输入,composer 主体预览当前匹配。Enter 接受预览,Esc 恢复搜索前的草稿。

### 提交与排队

**Enter**:立即提交。
**Tab**:任务运行中时排队;无任务时等同 Enter(确保输入不会丢失)。Tab 在输入 `!` shell 命令时不提交。

提交路径的处理流程:
1. 展开待处理的粘贴占位符,对齐元素范围
2. 裁剪空白并重定位文本元素
3. 清理本地附件图片(仅保留展开后存活的占位符)
4. 保留远程图片 URL 作为独立附件

关键设计:提交后清空 textarea 时,刻意保留 kill buffer。这让用户可以 Ctrl+K 剪切部分草稿,执行其他操作(如更改推理级别),然后 Ctrl+Y 粘贴回来。

### 远程图片行

远程图片 URL 渲染为不可编辑的 `[Image #N]` 行,位于 textarea 上方。支持的操作:

- Up 键在 textarea 光标位置 0 时进入远程行选择
- Up/Down 在远程行间移动
- Delete/Backspace 删除选中的远程图片
- 删除后自动重编号以保持连续性

### 斜杠命令

输入以 `/` 开头时触发斜杠命令弹窗。命令经过两阶段处理:

1. **暂存**:composer 将提交的斜杠文本暂存到本地历史
2. **记录**:ChatWidget 分发命令后正式记录(与普通提交文本一样)

带参数的命令(如 `/plan`、`/review`)复用相同的准备路径,保留粘贴内容和文本元素。

## 其他 UI 组件

### 状态栏

显示当前模型、token 使用量、速率限制状态等信息。通过 `StatusLineItem` 和 `StatusLineSetupView` 配置。

### Diff 渲染

`diff_render` 模块提供代码差异的富文本渲染,使用 `DiffSummary` 展示文件修改的摘要。在审批覆盖层中,用户可以查看命令将要做的更改。

### Markdown 渲染与流式处理

助手消息通常包含 Markdown 格式的文本。TUI 实现了流式 Markdown 渲染——在模型生成输出的同时实时渲染 Markdown 元素(标题、代码块、列表等)。

### 文件搜索

`FileSearchManager`(引用自 app.rs)和 `file_search_popup.rs` 提供模糊文件搜索功能。通过 `codex_file_search::FileMatch` 返回匹配结果。

### Exec Cell

命令执行的结果在 UI 中显示为特殊的执行单元格,包含命令文本、退出码和输出。

### Pager 覆盖层

`pager_overlay::Overlay` 提供全屏分页浏览功能,用于查看长输出或完整记录。

### ASCII 动画

`ascii_animation.rs` 提供终端动画效果,可能用于加载指示或其他视觉反馈。

### 桌面通知

通知系统检测终端焦点状态,根据 `NotificationCondition` 决定是否发送:

```rust
fn should_emit_notification(condition: NotificationCondition, terminal_focused: bool) -> bool {
    match condition {
        NotificationCondition::Unfocused => !terminal_focused,
        NotificationCondition::Always => true,
    }
}
```

## 数据流图

```mermaid
flowchart TD
    subgraph 输入层
        K[键盘输入] --> CS[crossterm EventStream]
        CS --> EB[EventBroker]
    end

    subgraph 事件分发
        EB --> TE[TuiEvent::Key / Paste / Draw]
        TE --> APP[App]
    end

    subgraph App 路由层
        APP -->|全局快捷键| GH[全局处理: 退出/中断]
        APP -->|聊天事件| CW[ChatWidget]
    end

    subgraph ChatWidget 层
        CW -->|Ctrl+T| TO[Transcript Overlay]
        CW -->|聊天事件| BP[BottomPane]
    end

    subgraph BottomPane 层
        BP -->|有模态视图| BPV[BottomPaneView 弹窗]
        BP -->|无模态视图| CC[ChatComposer]
    end

    subgraph ChatComposer 层
        CC -->|/ 斜杠命令| CP[Command Popup]
        CC -->|@ 提及| FS[File Search / Skill Mention]
        CC -->|Enter 提交| SUB[Submit]
        CC -->|Tab| QUE[Queue]
        CC -->|Ctrl+R| HS[History Search]
    end

    subgraph 提交处理
        SUB --> ASS[AppServerSession]
        QUE --> ASS
    end

    subgraph 服务层
        ASS -->|发送消息| CORE[codex-core]
        CORE -->|模型调用| API[LLM API]
        CORE -->|工具执行| EXEC[Exec System]
    end

    subgraph 响应回流
        API -->|流式响应| CORE
        EXEC -->|执行结果| CORE
        CORE -->|事件| ASS
        ASS -->|ServerNotification| APP
        APP -->|更新 UI| CW
        CW -->|更新单元格| RENDER[渲染循环]
    end

    subgraph 渲染
        RENDER --> FR[FrameRequester]
        FR --> TUI[Tui.terminal.draw]
        TUI --> SCREEN[终端屏幕]
    end

    style SCREEN fill:#6f6,color:#000
    style K fill:#69f,color:#fff
```

## 核心文件索引

| 文件 | 职责 |
|------|------|
| `codex-rs/tui/src/main.rs` | 入口: run_main(), 初始化流程, App::run() |
| `codex-rs/tui/src/tui.rs` | Tui 结构体: 终端管理, 模式设置, 事件分发 |
| `codex-rs/tui/src/app.rs` | App 结构体: 核心状态, 全局事件路由 |
| `codex-rs/tui/src/chatwidget.rs` | ChatWidget: 聊天界面, 历史单元格, 记录覆盖层 |
| `codex-rs/tui/src/bottom_pane/mod.rs` | BottomPane: 底部面板, 输入路由, 模态栈 |
| `codex-rs/tui/src/bottom_pane/chat_composer.rs` | ChatComposer: 文本编辑, 提交逻辑, 斜杠命令 |
| `codex-rs/tui/src/bottom_pane/chat_composer_history.rs` | 输入历史管理: 持久化 + 会话内历史 |
| `codex-rs/tui/src/bottom_pane/approval_overlay.rs` | 命令审批覆盖层 |
| `codex-rs/tui/src/bottom_pane/command_popup.rs` | 斜杠命令弹窗 |
| `codex-rs/tui/src/bottom_pane/file_search_popup.rs` | 文件搜索弹窗 |
| `codex-rs/tui/src/bottom_pane/bottom_pane_view.rs` | 模态视图基础 trait |
| `codex-rs/tui/src/app_event.rs` | AppEvent 事件类型定义 |
| `codex-rs/tui/src/app_command.rs` | AppCommand 命令类型定义 |
| `codex-rs/tui/src/app_server_session.rs` | AppServerSession: 与 app-server 通信 |
| `codex-rs/tui/src/history_cell.rs` | HistoryCell: 对话历史单元格 |
| `codex-rs/tui/src/diff_render.rs` | 差异渲染 |
| `codex-rs/tui/src/pager_overlay.rs` | 全屏分页覆盖层 |

## 设计洞察

### 为什么不用备用屏幕?

传统 TUI 应用(如 vim、htop)使用终端的备用屏幕(alternate screen)——切换到独立的缓冲区,退出时恢复原屏幕。Codex TUI 默认使用内联视口(inline viewport),对话历史保留在正常的滚动缓冲区中。这意味着用户可以在退出后向上滚动查看之前的对话。备用屏幕仅在全屏覆盖层(如 pager overlay)时使用。

### 分层事件消费

事件路由的分层设计允许每层独立处理自己关心的事件。这避免了一个巨大的事件处理函数,也使得添加新的 UI 组件变得简单——只需在合适的层次插入新的处理器。

Ctrl+C 的处理是这种设计的典型案例:同一个按键在不同上下文中有不同含义(关闭弹窗、取消搜索、中断任务、退出应用),分层处理优雅地解决了这种多义性。

### Kill Buffer 保留

ChatComposer 在提交后保留 kill buffer 看似是个小细节,但体现了对用户工作流的深入理解。用户经常在编辑中途需要执行操作(更改设置、查看状态),然后恢复编辑。保留 kill buffer 让 Ctrl+K/Ctrl+Y 的 Emacs 风格编辑体验不被打断。

### 双轨历史

持久化历史(跨会话)和本地历史(会话内)的分离是务实的选择。持久化历史只存文本(序列化简单),本地历史存完整状态(支持丰富的恢复)。回忆时根据来源选择恢复策略,在持久性和功能性之间取得平衡。

### 帧率控制

`FrameRequester` 和 `TARGET_FRAME_INTERVAL` 实现了节流绘制。当模型快速流式输出时,每个 token 都可能触发重绘。没有帧率限制,CPU 会被渲染消耗殆尽。帧率控制确保 UI 流畅的同时不浪费计算资源。
