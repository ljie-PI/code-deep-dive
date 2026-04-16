# 8. 会话与消息模型

## 8.1 存储对比

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **存储格式** | JSONL (追加写入) | SQLite (Drizzle ORM) | JSONL + SQLite (双存储) |
| **路径** | `~/.claude/projects/<sanitized-cwd>/<session-id>.jsonl` | `$XDG_DATA_HOME/opencode/opencode.db` | `~/.codex/sessions/rollout-<ts>-<uuid>.jsonl` + SQLite state DB |
| **消息链接** | `uuid` + `parentUuid` 链式 | `MessageID` (ULID, 自然有序) | `ThreadId` + `RolloutLine` |
| **会话恢复** | 重新解析 JSONL | 从 SQLite 查询 | 重读 JSONL + SQLite 索引 |
| **子 Agent 存储** | 独立 JSONL 文件 (`subagents/` 目录) | 同一数据库 (parentID 关联) | 独立线程记录 |
| **会话分支** | 无显式 fork (Worktree 提供分支能力) | `Session.fork()` + `parent_id` | 无 (线程模型, 可归档) |

## 8.2 消息/Entry 类型

### Claude Code
```
TranscriptMessage 类型:
├── user, assistant, attachment, system
├── bash_progress, powershell_progress, mcp_progress, sleep_progress (临时)
├── FileHistorySnapshotMessage (文件内容快照)
├── AttributionSnapshotMessage (归因状态)
├── ContextCollapseCommitEntry, ContextCollapseSnapshotEntry
├── ContentReplacementEntry
├── PersistedWorktreeSession
└── QueueOperationMessage
```

### OpenCode
```
数据库表:
├── SessionTable (id, project_id, workspace_id, parent_id, title, version, ...)
├── MessageTable (id, session_id, data: MessageV2.Info JSON blob)
├── PartTable (id, message_id, session_id, data: MessageV2.Part JSON blob)
├── TodoTable (session_id, content, status, priority, position)
└── SessionEntryTable (id, session_id, type, data)
```

### Codex
```
存储层:
├── Rollout JSONL (主要):
│   ├── SessionMetaLine (会话元数据头)
│   ├── RolloutItem / RolloutLine (会话事件)
│   ├── InitialHistory / ResumedHistory (历史恢复)
│   └── EventMsg (事件消息)
├── session_index.jsonl (name → ThreadId 映射)
└── SQLite state DB (元数据索引, backfill)
```

## 8.3 文件快照/版本控制

| 方面 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **机制** | `FileHistorySnapshot` 内嵌在 JSONL | 影子 Git 仓库 | Ghost Git 提交 |
| **位置** | Session JSONL 文件内 | `$XDG_DATA_HOME/opencode/snapshot/<project>/<hash>/` | 项目 Git 仓库 (不在任何分支上) |
| **操作** | 用于 undo/restore 文件变更 | `git add` + `git write-tree`, 可 `git diff` 恢复 | 不可见 git commit 捕获工作树状态 |
| **空间效率** | 低 (文件内容内嵌) | 高 (Git 去重) | 高 (Git 去重) |
| **配置** | `fileCheckpointingEnabled: true` (默认开启) | `snapshot` 配置字段 (可禁用) | 可配置大文件/目录忽略阈值 |

## 8.4 评价

| 维度 | Claude Code | OpenCode | Codex |
|------|-------------|----------|-------|
| **崩溃安全** | 高 (JSONL 每行完整 JSON) | 最高 (SQLite WAL) | 高 (JSONL + SQLite) |
| **查询灵活性** | 低 (需全文解析) | 最高 (SQL 查询) | 中 (SQLite 索引 + JSONL 全文) |
| **空间效率** | 低 (JSONL + 内嵌快照) | 高 (SQLite + Git 快照) | 中 (双存储) |
| **会话分支** | 无 | 最佳 (DB fork) | 无 |
| **多端访问** | 不支持 | 支持 (SQLite 并发) | 有限 |
