# 第十五章：安全、审批与沙箱隔离

> **一句话总结：** Hermes-Agent 通过多层纵深防御体系——危险命令审批、路径遍历防护、URL/网站策略、凭证保护、技能扫描、敏感数据脱敏、网关配对认证、ACP 权限桥接以及容器沙箱隔离——构建了一个从用户输入到命令执行全链路覆盖的安全模型。

---

## 15.1 安全架构总览

Hermes-Agent 的安全体系并非单一网关，而是由多个独立但协同工作的安全层组成的纵深防御（Defense in Depth）架构。每一层针对不同的威胁向量，任何单一层被绕过时，后续层仍能提供保护。

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart TD
    subgraph 用户输入
        A[用户命令/请求]
    end

    subgraph 预执行安全检查
        B[Tirith 内容安全扫描]
        C[危险命令模式检测]
        D[URL 安全性验证]
        E[网站策略检查]
    end

    subgraph 审批系统
        F[Smart Approval / LLM 风险评估]
        G[用户交互审批 CLI/Gateway]
        H[会话级/永久级白名单]
    end

    subgraph 资源安全
        I[路径遍历防护]
        J[凭证文件保护]
        K[技能安全扫描]
        L[OSV 恶意包检测]
    end

    subgraph 运行时保护
        M[容器沙箱隔离]
        N[敏感数据脱敏]
        O[网关配对认证]
        P[ACP 权限桥接]
    end

    A --> B & C & D & E
    B --> F
    C --> F
    F -->|不确定| G
    F -->|自动批准| H
    G --> H
    D --> M
    E --> M
    I --> M
    J --> M
    K --> M
    L --> M
    M --> N
    O --> A
    P --> G
```

### 安全层一览

| 安全层 | 核心文件 | 防御目标 |
|--------|----------|----------|
| 危险命令审批 | `tools/approval.py` | 阻止 `rm -rf /`、`DROP TABLE` 等破坏性命令 |
| Tirith 安全扫描 | `tools/tirith_security.py` | 检测同形文字 URL、终端注入、管道注入等内容级威胁 |
| 路径安全 | `tools/path_security.py` | 防止路径遍历攻击 |
| URL 安全 | `tools/url_safety.py` | 阻止 SSRF 攻击（内网地址、云元数据端点） |
| 网站策略 | `tools/website_policy.py` | 用户可配置的域名黑名单 |
| 凭证保护 | `tools/credential_files.py` | 防止凭证文件逃逸到容器外 |
| 技能安全 | `tools/skills_guard.py` | 静态分析外部技能的恶意代码模式 |
| 数据脱敏 | `agent/redact.py` | 日志和输出中的 API 密钥、令牌等自动脱敏 |
| 网关认证 | `gateway/pairing.py` | 消息平台用户的配对码认证 |
| ACP 权限 | `acp_adapter/permissions.py` | ACP 协议的权限请求桥接 |
| OSV 检测 | `tools/osv_check.py` | MCP 扩展包的恶意软件检测 |
| 容器沙箱 | `tools/environments/docker.py` | Docker/Singularity 的安全隔离 |

---

## 15.2 审批系统（Approval System）

### 15.2.1 危险命令检测

审批系统的核心位于 `tools/approval.py`，它是危险命令检测、审批状态管理和白名单持久化的唯一事实来源（single source of truth）。

**检测机制：** 系统维护了一份包含 30+ 个正则表达式的 `DANGEROUS_PATTERNS` 列表（`tools/approval.py:75-133`），覆盖以下威胁类别：

| 类别 | 示例模式 | 描述 |
|------|----------|------|
| 文件系统破坏 | `rm -rf /`, `find -delete` | 递归删除、格式化文件系统 |
| 数据库操作 | `DROP TABLE`, `DELETE FROM`（无 WHERE） | SQL 破坏性操作 |
| 系统配置 | `> /etc/`, `sed -i /etc/` | 覆写系统配置文件 |
| 远程代码执行 | `curl \| bash`, `python -c` | 管道到解释器、脚本执行 |
| Git 破坏性操作 | `git reset --hard`, `git push --force` | 丢失未提交更改、重写历史 |
| 进程管理 | `kill -9 -1`, `pkill hermes` | 杀死所有进程、自我终止保护 |
| 权限滥用 | `chmod 777`, `chown -R root` | 全局可写权限 |
| 敏感路径写入 | `tee ~/.ssh/`, `>> ~/.hermes/.env` | 写入 SSH 密钥、环境变量文件 |

**反绕过措施：** 检测之前会对命令进行规范化处理（`tools/approval.py:163-178`）：
1. 剥离 ANSI 转义序列（完整 ECMA-48 规范）
2. 移除空字节（`\x00`）
3. Unicode NFKC 规范化（防止全角字符绕过）

```python
# tools/approval.py:181-192
def detect_dangerous_command(command: str) -> tuple:
    command_lower = _normalize_command_for_detection(command).lower()
    for pattern, description in DANGEROUS_PATTERNS:
        if re.search(pattern, command_lower, re.IGNORECASE | re.DOTALL):
            pattern_key = description
            return (True, pattern_key, description)
    return (False, None, None)
```

### 15.2.2 审批流程

整个审批流程由 `check_all_command_guards()` 函数编排（`tools/approval.py:690-919`），它合并了 Tirith 安全扫描和危险命令检测的结果，呈现为单一审批请求：

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart TD
    CMD[命令输入] --> ENV{容器环境?}
    ENV -->|docker/singularity/modal| ALLOW[直接放行]
    ENV -->|local/ssh| YOLO{YOLO 模式?}
    YOLO -->|是| ALLOW
    YOLO -->|否| DETECT[运行检测]

    DETECT --> TIRITH[Tirith 扫描]
    DETECT --> PATTERN[模式匹配]

    TIRITH --> MERGE{汇总发现}
    PATTERN --> MERGE

    MERGE -->|无警告| ALLOW
    MERGE -->|有警告| APPROVED{已批准?}
    APPROVED -->|是| ALLOW
    APPROVED -->|否| MODE{审批模式}

    MODE -->|smart| LLM[LLM 风险评估]
    MODE -->|manual| PROMPT[用户提示]

    LLM -->|approve| ALLOW
    LLM -->|deny| BLOCK[阻止]
    LLM -->|escalate| PROMPT

    PROMPT --> CHOICE{用户选择}
    CHOICE -->|once| ALLOW
    CHOICE -->|session| SESSION[会话白名单]
    CHOICE -->|always| PERM[永久白名单]
    CHOICE -->|deny/timeout| BLOCK

    SESSION --> ALLOW
    PERM --> ALLOW
```

**关键设计决策：**

1. **容器环境自动放行：** 当 `env_type` 为 `docker`、`singularity`、`modal` 或 `daytona` 时，所有检查被跳过（`tools/approval.py:598-599`）。理由是容器本身就是沙箱，破坏性命令只影响容器内部。

2. **三级审批范围：**
   - `once`：仅本次命令
   - `session`：当前会话内同类模式自动通过
   - `always`：写入 `config.yaml` 的永久白名单

3. **Smart Approval（智能审批）：** 当 `approvals.mode=smart` 时，系统调用辅助 LLM 进行风险评估（`tools/approval.py:531-580`），将模式匹配的误报率降至最低。LLM 返回 `APPROVE`/`DENY`/`ESCALATE`，只有明确安全时才自动通过，不确定时回退到人工审批。

4. **Tirith 的特殊处理：** Tirith 发现的安全问题不允许 `always`（永久白名单），只能 `session` 级别（`tools/approval.py:895`），因为内容级安全发现不适合进行宽泛的永久豁免。

### 15.2.3 会话状态管理

审批状态通过线程安全的内存数据结构管理（`tools/approval.py:198-204`），支持网关场景下的并发会话：

- **`_session_approved`：** 每会话已批准的模式集合（`dict[str, set]`）
- **`_session_yolo`：** 启用 YOLO 旁路的会话集合
- **`_permanent_approved`：** 跨进程持久化的永久白名单
- **`_gateway_queues`：** 网关阻塞式审批队列（每请求一个 `threading.Event`）

网关审批使用基于队列的阻塞机制（`tools/approval.py:213-279`）：每个待审批命令创建一个 `_ApprovalEntry`，agent 线程在 `Event.wait()` 上阻塞，直到用户通过 `/approve` 或 `/deny` 解除。`/approve all` 可以一次性解除所有待审批项。

---

## 15.3 路径安全（Path Security）

路径安全模块 `tools/path_security.py` 提供两个核心防护函数：

**1. `validate_within_dir(path, root)`** — 确保路径在允许目录内（`tools/path_security.py:15-34`）：

```python
def validate_within_dir(path: Path, root: Path) -> Optional[str]:
    resolved = path.resolve()          # 解析符号链接 + 规范化 ..
    root_resolved = root.resolve()
    resolved.relative_to(root_resolved) # 如果不在 root 下则抛出 ValueError
```

此函数使用 `Path.resolve()` 跟随符号链接并规范化 `..` 组件，然后通过 `relative_to()` 验证解析后的路径确实位于根目录内。这是抵御路径遍历攻击的核心防线。

**2. `has_traversal_component(path_str)`** — 快速检测明显的遍历尝试（`tools/path_security.py:37-43`），在完整路径解析之前进行预过滤。

**调用点分析：** `validate_within_dir` 被多个模块调用，形成统一的路径安全层：
- `tools/credential_files.py:85` — 凭证文件注册时防止 `../../.ssh/id_rsa` 式逃逸
- `tools/credential_files.py:154` — 配置文件中声明的凭证路径同样受限
- 技能管理工具、定时任务工具等均依赖此函数

---

## 15.4 URL 安全（URL Safety）

### 15.4.1 SSRF 防护

`tools/url_safety.py` 防止 SSRF（Server-Side Request Forgery）攻击，阻止 Agent 被恶意提示诱导访问内网资源。

**阻止的地址范围（`tools/url_safety.py:39-48`）：**

| 地址类型 | 示例 | 防御目标 |
|----------|------|----------|
| 私有地址 | `10.0.0.0/8`, `192.168.0.0/16` | 内网服务 |
| 回环地址 | `127.0.0.1`, `::1` | 本机服务 |
| 链路本地 | `169.254.169.254` | 云元数据端点（AWS/GCP/Azure） |
| CGNAT | `100.64.0.0/10` | Tailscale/WireGuard VPN、运营商级 NAT |
| 组播/未指定 | `224.0.0.0/4`, `0.0.0.0` | 特殊地址 |
| 已知内部主机名 | `metadata.google.internal` | 显式阻止的 GCP 元数据主机名 |

**设计原则：失败关闭（fail-closed）。** DNS 解析失败或任何意外异常都会导致请求被阻止（`tools/url_safety.py:72-74`, `93-97`）。这意味着 SSRF 绕过尝试无法利用 DNS 错误来逃逸检查。

**已知限制（文档中明确记录）：**
- **DNS 重绑定攻击：** 攻击者控制的 DNS 服务器可以在检查时返回公网 IP，在实际连接时返回私有 IP。修复需要连接级验证或出口代理。
- **重定向绕过：** 通过 httpx 事件钩子在每次重定向时重新验证目标地址来缓解。

### 15.4.2 网站策略（Website Policy）

`tools/website_policy.py` 实现了用户可配置的域名黑名单系统。策略从 `~/.hermes/config.yaml` 的 `security.website_blocklist` 字段加载。

**配置结构：**

```yaml
security:
  website_blocklist:
    enabled: true
    domains:
      - "malicious-site.com"
      - "*.phishing-domain.org"
    shared_files:
      - "/path/to/corporate-blocklist.txt"
```

**匹配规则（`tools/website_policy.py:209-214`）：**
- 精确匹配：`example.com` 匹配 `example.com` 和 `sub.example.com`
- 通配符匹配：`*.example.com` 使用 `fnmatch` 进行 glob 匹配
- 自动规范化：移除 `www.` 前缀、协议头、尾部点号

**性能优化：** 策略结果在内存中缓存 30 秒（`tools/website_policy.py:35`），避免每次 URL 检查都重新解析 YAML。设计为**失败开放（fail-open）**——配置文件错误不会导致所有 Web 工具失效。

---

## 15.5 凭证文件保护

`tools/credential_files.py` 的核心职责是在远程终端后端（Docker、Modal、SSH）中安全挂载凭证文件，同时防止恶意技能通过凭证声明逃逸宿主文件系统。

### 安全控制点

**1. 路径遍历防护（`tools/credential_files.py:55-102`）：**

```python
def register_credential_file(relative_path, container_base="/root/.hermes"):
    # 拒绝绝对路径
    if os.path.isabs(relative_path):
        return False
    # 路径必须在 HERMES_HOME 内
    containment_error = validate_within_dir(host_path, hermes_home)
    if containment_error:
        return False
```

恶意技能声明 `required_credential_files: ['../../.ssh/id_rsa']` 将被拒绝。

**2. 符号链接安全（`tools/credential_files.py:249-289`）：**

技能目录挂载到容器时，如果检测到符号链接，系统不会直接绑定挂载（因为 Docker bind mount 会跟随符号链接，暴露任意宿主文件），而是创建一个净化后的临时副本（仅包含常规文件），然后挂载该副本。

**3. 会话隔离（`tools/credential_files.py:33`）：** 凭证注册使用 `ContextVar` 实现，防止网关多会话场景下的跨会话数据泄漏。

---

## 15.6 技能安全扫描（Skills Guard）

`tools/skills_guard.py` 是一个综合性的静态分析安全扫描器，所有从注册表下载的技能在安装前必须通过扫描。

### 15.6.1 信任等级模型

```mermaid
%%{init: {'theme': 'neutral'}}%%
flowchart LR
    subgraph 信任等级
        BUILTIN[builtin<br/>内置技能]
        TRUSTED[trusted<br/>可信源]
        COMMUNITY[community<br/>社区源]
        AGENT[agent-created<br/>Agent 创建]
    end

    subgraph 安装策略
        SAFE[safe 判定]
        CAUTION[caution 判定]
        DANGEROUS[dangerous 判定]
    end

    BUILTIN ---|全部允许| SAFE & CAUTION & DANGEROUS
    TRUSTED ---|允许| SAFE & CAUTION
    TRUSTED ---|阻止| DANGEROUS
    COMMUNITY ---|允许| SAFE
    COMMUNITY ---|阻止| CAUTION & DANGEROUS
    AGENT ---|允许| SAFE & CAUTION
    AGENT ---|需确认| DANGEROUS
```

信任等级与判定结果的安装策略矩阵（`tools/skills_guard.py:41-47`）：

| 信任等级 | safe | caution | dangerous |
|----------|------|---------|-----------|
| builtin | allow | allow | allow |
| trusted | allow | allow | block |
| community | allow | block | block |
| agent-created | allow | allow | ask |

可信源仅限 `openai/skills` 和 `anthropics/skills` 两个硬编码仓库（`tools/skills_guard.py:39`）。

### 15.6.2 威胁检测模式

扫描器包含 80+ 个正则表达式模式（`tools/skills_guard.py:82-484`），覆盖 12 个威胁类别：

| 类别 | 数量 | 严重程度 | 示例 |
|------|------|----------|------|
| **exfiltration** | ~20 | critical/high | `curl $SECRET`, `cat .env`, `printenv` |
| **injection** | ~15 | critical/high | `ignore previous instructions`, DAN 越狱 |
| **destructive** | ~8 | critical | `rm -rf /`, `mkfs`, `dd if=` |
| **persistence** | ~10 | medium/critical | `crontab`, `authorized_keys`, `systemd enable` |
| **network** | ~10 | critical/high | 反向 shell、`ngrok`、webhook 服务 |
| **obfuscation** | ~15 | high/medium | `base64 -d \| bash`, `eval("...")`、chr 构建 |
| **supply_chain** | ~8 | critical/medium | `curl \| bash`、未钉版本的 pip install |
| **traversal** | ~5 | high/critical | `../../../`, `/etc/passwd` |
| **credential_exposure** | ~6 | critical | 硬编码 API 密钥、嵌入的私钥 |
| **privilege_escalation** | ~5 | high/critical | `sudo`、`setuid`、`NOPASSWD` |
| **mining** | ~2 | critical/medium | `xmrig`、`stratum+tcp` |
| **execution** | ~6 | high/medium | `subprocess.run`、`os.system` |

### 15.6.3 结构检查

除了模式扫描，还执行目录结构分析（`tools/skills_guard.py:734-848`）：
- 文件数量上限：50 个（`MAX_FILE_COUNT`）
- 总大小上限：1MB（`MAX_TOTAL_SIZE_KB`）
- 单文件上限：256KB（`MAX_SINGLE_FILE_KB`）
- 二进制文件检测：`.exe`、`.dll`、`.so` 等不应出现在技能中
- 符号链接逃逸：必须解析在技能目录内
- 不可见 Unicode 字符检测：零宽空格、RTL 覆盖等 18 种注入字符（`tools/skills_guard.py:505-523`）

---

## 15.7 敏感数据脱敏（Redaction）

`agent/redact.py` 在日志和输出中自动脱敏敏感数据。

### 脱敏策略

**安全设计：** 脱敏启用状态在模块导入时快照（`agent/redact.py:18`），运行时环境变量修改（如 LLM 生成的 `export HERMES_REDACT_SECRETS=false`）无法禁用脱敏。

**已知前缀模式（`agent/redact.py:21-57`）：** 系统识别 30+ 种 API 密钥前缀：

| 前缀 | 服务 |
|------|------|
| `sk-` | OpenAI / Anthropic |
| `ghp_`, `github_pat_` | GitHub PAT |
| `xox[baprs]-` | Slack |
| `AIza` | Google API |
| `AKIA` | AWS Access Key |
| `sk_live_`, `sk_test_` | Stripe |
| `tvly-` | Tavily |
| 等 | 等 |

**脱敏覆盖的数据类型：**
1. **已知前缀令牌** — 通过正则匹配
2. **环境变量赋值** — `OPENAI_API_KEY=sk-abc...` 形式
3. **JSON 字段值** — `"apiKey": "value"` 形式
4. **Authorization 头** — `Bearer` 令牌
5. **Telegram Bot 令牌** — `bot<digits>:<token>` 格式
6. **私钥块** — `-----BEGIN PRIVATE KEY-----` 整块替换
7. **数据库连接字符串密码** — `postgres://user:PASSWORD@host` 中的密码
8. **E.164 电话号码** — Signal/WhatsApp 等平台的电话号码

**脱敏规则：** 短令牌（<18 字符）完全替换为 `***`；长令牌保留前 6 位和后 4 位以便调试（`agent/redact.py:106-110`）。

**日志集成：** `RedactingFormatter` 类（`agent/redact.py:173-181`）作为 Python `logging.Formatter` 的子类，自动对所有日志消息应用脱敏。

---

## 15.8 Tirith 安全扫描

`tools/tirith_security.py` 是 Tirith 二进制文件的包装器，提供命令执行前的内容级安全扫描。

### 架构设计

**退出码即判定：** Tirith 使用退出码作为判定源（`tools/tirith_security.py:7-8`）：
- `0` = 放行（allow）
- `1` = 阻止（block）
- `2` = 警告（warn）

JSON 输出仅用于丰富发现详情，绝不覆盖退出码判定。

**失败模式（`tools/tirith_security.py:617-652`）：**
- 进程启动失败（`OSError`）：尊重 `fail_open` 配置
- 超时：尊重 `fail_open` 配置（默认 5 秒）
- 未知退出码：尊重 `fail_open` 配置

**自动安装（`tools/tirith_security.py:281-371`）：**
Tirith 不在 PATH 上时，自动从 GitHub Releases 下载安装到 `$HERMES_HOME/bin/tirith`：
1. 始终验证 SHA-256 校验和
2. 如果 cosign 可用，额外验证 GitHub Actions 工作流签名（供应链溯源）
3. 如果 cosign 不可用，仅 SHA-256 + HTTPS 验证（仍然安全，只是缺少供应链溯源证明）
4. 安装失败结果持久化到磁盘（24 小时内不重试），避免频繁网络请求

---

## 15.9 网关配对认证（Gateway Pairing）

`gateway/pairing.py` 实现了基于一次性配对码的消息平台用户认证系统。

### 安全特性

系统遵循 OWASP 和 NIST SP 800-63-4 指南（`gateway/pairing.py:8-17`）：

| 安全特性 | 实现 | 代码位置 |
|----------|------|----------|
| 密码学随机 | `secrets.choice()` | `gateway/pairing.py:178` |
| 无歧义字母表 | 32 字符（排除 0/O、1/I） | `gateway/pairing.py:33` |
| 配对码长度 | 8 字符 | `gateway/pairing.py:34` |
| 码过期时间 | 1 小时 | `gateway/pairing.py:37` |
| 平台待配额 | 每平台最多 3 个待审配对码 | `gateway/pairing.py:42` |
| 速率限制 | 每用户每 10 分钟 1 次 | `gateway/pairing.py:38` |
| 失败锁定 | 5 次失败后锁定 1 小时 | `gateway/pairing.py:43-44` |
| 文件权限 | `chmod 0o600` | `gateway/pairing.py:64` |
| 原子写入 | temp + rename 模式 | `gateway/pairing.py:49-72` |

### 配对流程

```mermaid
%%{init: {'theme': 'neutral'}}%%
sequenceDiagram
    participant U as 未知用户
    participant GW as Gateway
    participant PS as PairingStore
    participant Owner as 机器人所有者

    U->>GW: 发送消息
    GW->>PS: is_approved()?
    PS-->>GW: False
    GW->>PS: generate_code()
    PS-->>GW: "K3HN7V2M"
    GW->>U: "请让管理员批准配对码"
    GW->>Owner: "[CLI] 新配对请求: K3HN7V2M"
    Owner->>PS: approve_code("K3HN7V2M")
    PS-->>Owner: {user_id, user_name}
    Note over PS: 用户加入已批准列表
    U->>GW: 后续消息
    GW->>PS: is_approved()?
    PS-->>GW: True
    GW->>U: 正常响应
```

---

## 15.10 ACP 安全

### 15.10.1 认证

`acp_adapter/auth.py` 提供轻量级的运行时提供者检测（`acp_adapter/auth.py:8-19`）。它通过 `resolve_runtime_provider()` 获取当前配置的 API 密钥和提供者信息，验证两者都存在且非空。

### 15.10.2 权限桥接

`acp_adapter/permissions.py` 将 ACP 协议的权限请求映射到 Hermes 的审批回调机制（`acp_adapter/permissions.py:26-77`）。

**映射关系（`acp_adapter/permissions.py:18-23`）：**

| ACP PermissionOptionKind | Hermes 审批结果 |
|--------------------------|----------------|
| `allow_once` | `once` |
| `allow_always` | `always` |
| `reject_once` | `deny` |
| `reject_always` | `deny` |

**桥接机制：** `make_approval_callback()` 返回一个同步的 Hermes 审批回调函数，内部通过 `asyncio.run_coroutine_threadsafe()` 桥接到 ACP 的异步权限请求。超时或失败时默认拒绝（`acp_adapter/permissions.py:62-64`）。

---

## 15.11 容器沙箱隔离

### Docker 安全加固

`tools/environments/docker.py` 中的 Docker 后端应用了全面的安全加固（`tools/environments/docker.py:136-145`）：

```python
"--cap-drop", "ALL",              # 丢弃所有 Linux 能力
"--cap-add", "DAC_OVERRIDE",      # 仅保留必要能力
"--cap-add", "CHOWN",
"--cap-add", "FOWNER",
"--security-opt", "no-new-privileges",  # 禁止提权
"--pids-limit", "256",            # 进程数限制（防 fork 炸弹）
"--tmpfs", "/tmp:rw,nosuid,size=512m",      # noexec 临时文件系统
"--tmpfs", "/var/tmp:rw,noexec,nosuid,size=256m",
"--tmpfs", "/run:rw,noexec,nosuid,size=64m",
```

**安全配置要点：**

| 措施 | 作用 |
|------|------|
| `--cap-drop ALL` | 丢弃所有 Linux 能力，仅保留 3 个必需能力 |
| `--no-new-privileges` | 容器内进程无法通过 `setuid`/`setgid` 获取新权限 |
| `--pids-limit 256` | 限制进程数，防止 fork 炸弹 |
| `tmpfs` 的 `nosuid`/`noexec` | 防止在临时目录中执行可执行文件或利用 setuid |
| 可配置的 CPU/内存/磁盘限制 | 通过 `--cpus`、`--memory`、`--storage-opt` 参数 |

### Singularity 安全加固

`tools/environments/singularity.py` 使用 `--containall`（完全隔离，无宿主挂载）和 `--no-home`（不挂载用户主目录），并支持能力丢弃。

### 审批系统的容器感知

最关键的设计是审批系统对容器环境的感知：当命令在容器内执行时（`env_type` 为 `docker`/`singularity`/`modal`/`daytona`），所有危险命令检查和 Tirith 扫描被完全旁路（`tools/approval.py:598-599`, `700-701`）。这是合理的安全权衡——容器提供了物理隔离，容器内的破坏性操作不影响宿主系统。

---

## 15.12 OSV 恶意软件检测

`tools/osv_check.py` 在通过 `npx`/`uvx` 启动 MCP 服务器扩展之前，查询 Google 的 OSV（Open Source Vulnerabilities）API 检测已知恶意软件。

**关键设计（`tools/osv_check.py:26-62`）：**
- **仅检测恶意软件：** 只关注 `MAL-*` 前缀的公告 ID，忽略常规 CVE
- **失败开放：** 网络错误、超时、解析失败均允许包继续加载
- **生态系统推断：** 从命令名（`npx` → npm，`uvx`/`pipx` → PyPI）自动推断包生态系统
- **版本感知：** 解析 `@scope/name@version`（npm）和 `name==version`（PyPI）格式

---

## 15.13 关键文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `tools/approval.py` | ~924 | 危险命令检测、审批流程编排、会话/永久白名单管理 |
| `tools/path_security.py` | ~44 | 路径遍历防护（`validate_within_dir`、`has_traversal_component`） |
| `tools/tirith_security.py` | ~671 | Tirith 二进制包装器：自动安装、SHA-256/cosign 验证、命令扫描 |
| `tools/url_safety.py` | ~98 | SSRF 防护：私有 IP 检测、DNS 解析、fail-closed |
| `tools/website_policy.py` | ~283 | 域名黑名单策略：配置加载、缓存、通配符匹配 |
| `tools/credential_files.py` | ~407 | 凭证文件安全挂载：路径验证、符号链接净化、会话隔离 |
| `tools/skills_guard.py` | ~978 | 技能安全扫描器：80+ 威胁模式、信任等级策略、结构检查 |
| `agent/redact.py` | ~182 | 30+ 种 API 密钥脱敏、日志格式化器集成 |
| `gateway/pairing.py` | ~310 | 配对码认证：密码学随机、速率限制、失败锁定 |
| `acp_adapter/auth.py` | ~25 | ACP 提供者检测 |
| `acp_adapter/permissions.py` | ~78 | ACP 权限到 Hermes 审批的桥接 |
| `tools/osv_check.py` | ~156 | MCP 扩展恶意软件检测（OSV API） |
| `tools/environments/docker.py` | ~280+ | Docker 安全加固：cap-drop、no-new-privileges、资源限制 |
| `tools/environments/singularity.py` | ~60+ | Singularity 安全加固：containall、no-home |

---

## 15.14 安全模型设计原则总结

1. **纵深防御：** 没有任何单一安全层承担所有保护职责。即使审批系统被绕过，容器沙箱仍然提供隔离；即使路径验证被绕过，符号链接净化仍然防止文件逃逸。

2. **最小权限：** Docker 容器丢弃所有能力后仅添加 3 个必需能力；凭证文件仅允许相对路径且必须在 HERMES_HOME 内。

3. **失败模式明确：** 每个安全组件都明确定义了失败行为——URL 安全 fail-closed（宁可误拒），网站策略 fail-open（不因配置错误瘫痪功能），Tirith 可配置（默认 fail-open）。

4. **反绕过意识：** 命令检测前进行 ANSI 剥离、null byte 移除和 Unicode 规范化；脱敏状态在导入时快照防止运行时篡改；配对码使用密码学随机且排除易混淆字符。

5. **分层审批范围：** once/session/always 三级范围让用户在安全性和便利性之间取得平衡，同时 Tirith 发现限制在 session 级，防止不当的永久豁免。
