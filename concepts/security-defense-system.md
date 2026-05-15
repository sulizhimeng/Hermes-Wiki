---
title: 安全防御体系 — 多层注入检测
created: 2026-04-07
updated: 2026-05-15
type: concept
tags: [architecture, security, injection-defense, skills-guard, supply-chain]
sources: [hermes-agent 源码分析 2026-04-07, 2026-05-15 增量验证]
---

# 安全防御体系 — 多层注入检测

## 设计原理

Hermes Agent 具有执行代码、读写文件、访问网络的能力，因此必须防御：
1. **提示注入** — 恶意内容试图覆盖 Agent 指令
2. **数据泄露** — 窃取 API 密钥、凭证
3. **破坏性操作** — 删除文件、破坏系统
4. **持久化后门** — 修改启动脚本、cron 任务
5. **供应链攻击** — 恶意技能、未锁定依赖

Hermes 实现了 **5 层防御体系**，从内容扫描到信任策略。

## 第 1 层：Skills Guard 安全扫描

### 威胁模式库（100+ 正则模式）

```python
THREAT_PATTERNS = [
    # ── 数据泄露 ──
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD)',
     "env_exfil_curl", "critical", "exfiltration",
     "curl 命令插值密钥环境变量"),
    (r'os\.getenv\s*\(\s*[^\)]*(?:KEY|TOKEN|SECRET|PASSWORD)',
     "python_getenv_secret", "critical", "exfiltration",
     "通过 os.getenv() 读取密钥"),
    (r'\$HOME/\.ssh|\~/\.ssh',
     "ssh_dir_access", "high", "exfiltration",
     "引用用户 SSH 目录"),
    (r'\$HOME/\.hermes/\.env|\~/\.hermes/\.env',
     "hermes_env_access", "critical", "exfiltration",
     "直接引用 Hermes 密钥文件"),
    
    # ── 提示注入 ──
    (r'ignore\s+(?:\w+\s+)*(previous|all|above|prior)\s+instructions',
     "prompt_injection_ignore", "critical", "injection",
     "提示注入：忽略之前指令"),
    (r'do\s+not\s+(?:\w+\s+)*tell\s+(?:\w+\s+)*the\s+user',
     "deception_hide", "critical", "injection",
     "指示 Agent 向用户隐藏信息"),
    (r'act\s+as\s+(if|though)\s+(?:\w+\s+)*you\s+(?:\w+\s+)*(have\s+no|don\'t\s+have)\s+(?:\w+\s+)*(restrictions|limits|rules)',
     "bypass_restrictions", "critical", "injection",
     "指示 Agent 无限制地行动"),
    
    # ── 破坏性操作 ──
    (r'rm\s+-rf\s+/',
     "destructive_root_rm", "critical", "destructive",
     "从根目录递归删除"),
    (r'shutil\.rmtree\s*\(\s*[\"\']/',
     "python_rmtree", "high", "destructive",
     "Python rmtree 绝对路径"),
    (r'>\s*/etc/',
     "system_overwrite", "critical", "destructive",
     "覆盖系统配置文件"),
    
    # ── 持久化后门 ──
    (r'\bcrontab\b',
     "persistence_cron", "medium", "persistence",
     "修改 cron 任务"),
    (r'authorized_keys',
     "ssh_backdoor", "critical", "persistence",
     "修改 SSH 授权密钥"),
    (r'systemd.*\.service|systemctl\s+(enable|start)',
     "systemd_service", "medium", "persistence",
     "引用或启用 systemd 服务"),
    (r'\.(bashrc|zshrc|profile)',
     "shell_rc_mod", "medium", "persistence",
     "引用 shell 启动文件"),
    
    # ── 网络后门 ──
    (r'\bnc\s+-[lp]|ncat\s+-[lp]|\bsocat\b',
     "reverse_shell", "critical", "network",
     "潜在反向 shell 监听器"),
    (r'\bngrok\b|\blocaltunnel\b|\bserveo\b',
     "tunnel_service", "high", "network",
     "使用隧道服务获取外部访问"),
    
    # ── 混淆执行 ──
    (r'base64\s+(-d|--decode)\s*\|',
     "base64_decode_pipe", "high", "obfuscation",
     "base64 解码并管道执行"),
    (r'\beval\s*\(\s*[\"\']',
     "eval_string", "high", "obfuscation",
     "eval() 字符串参数"),
    (r'echo\s+[^\n]*\|\s*(bash|sh|python)',
     "echo_pipe_exec", "critical", "obfuscation",
     "echo 管道解释器执行"),
    
    # ── 供应链攻击 ──
    (r'curl\s+[^\n]*\|\s*(ba)?sh',
     "curl_pipe_shell", "critical", "supply_chain",
     "curl 管道到 shell（下载执行）"),
    (r'pip\s+install\s+(?!-r\s)(?!.*==)',
     "unpinned_pip_install", "medium", "supply_chain",
     "pip install 无版本锁定"),
    
    # ── 硬编码密钥 ──
    (r'(?:api[_-]?key|token|secret|password)\s*[=:]\s*[\"\'][A-Za-z0-9+/=_-]{20,}',
     "hardcoded_secret", "critical", "credential_exposure",
     "可能的硬编码 API 密钥"),
    (r'-----BEGIN\s+(RSA\s+)?PRIVATE\s+KEY-----',
     "embedded_private_key", "critical", "credential_exposure",
     "嵌入的私钥"),
]
```

### 不可见 Unicode 检测

```python
INVISIBLE_CHARS = {
    '\u200b',  # 零宽空格
    '\u200c',  # 零宽非连接符
    '\u200d',  # 零宽连接符
    '\u2060',  # 词连接符
    '\ufeff',  # 零宽不换行空格 (BOM)
    '\u202a',  # 从左到右嵌入
    '\u202b',  # 从右到左嵌入
    '\u202e',  # 从右到左覆盖
    # ... 共 17 个字符
}

# 检测技能文件中的不可见字符
for i, line in enumerate(lines, start=1):
    for char in INVISIBLE_CHARS:
        if char in line:
            findings.append(Finding(
                pattern_id="invisible_unicode",
                severity="high",
                category="injection",
                match=f"U+{ord(char):04X}",
                description="不可见 Unicode 字符（可能文本隐藏/注入）",
            ))
```

### 结构检查

```python
MAX_FILE_COUNT = 50       # 技能不应有 50+ 文件
MAX_TOTAL_SIZE_KB = 1024  # 总大小 1MB 可疑
MAX_SINGLE_FILE_KB = 256  # 单文件 > 256KB 可疑

SUSPICIOUS_BINARY_EXTENSIONS = {
    '.exe', '.dll', '.so', '.dylib', '.bin',
    '.msi', '.dmg', '.app', '.deb', '.rpm',
    '.dat', '.com',
}
```

## 第 2 层：信任级别策略

```python
TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

INSTALL_POLICY = {
    #               safe      caution    dangerous
    "builtin":     ("allow",  "allow",   "allow"),
    "trusted":     ("allow",  "allow",   "block"),
    "community":   ("allow",  "block",   "block"),
    "agent-created":("allow", "allow",   "ask"),
}

VERDICT_INDEX = {"safe": 0, "caution": 1, "dangerous": 2}
```

### 裁决逻辑

```python
def _determine_verdict(findings):
    if not findings:
        return "safe"
    
    has_critical = any(f.severity == "critical" for f in findings)
    has_high = any(f.severity == "high" for f in findings)
    
    if has_critical:
        return "dangerous"
    if has_high:
        return "caution"
    return "safe"  # 仅 medium/low
```

### 安装决策

| 来源 | safe | caution | dangerous |
|------|------|---------|-----------|
| builtin（内置） | allow | allow | allow |
| trusted（OpenAI/Anthropic） | allow | allow | block |
| community（社区） | allow | **block** | block |
| agent-created（Agent 创建） | allow | allow | **ask** |

## 第 3 层：Memory 内容扫描

```python
_MEMORY_THREAT_PATTERNS = [
    # 提示注入
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'act\s+as\s+(if|though)\s+.*no\s+(restrictions|limits)', "bypass_restrictions"),
    # 密钥泄露
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials|\.netrc)', "read_secrets"),
    (r'base64\s+(-d|--decode)\s*\|', "base64_decode_pipe"),
    # 持久化后门
    (r'authorized_keys', "ssh_backdoor"),
    (r'\$HOME/\.ssh|\~/\.ssh', "ssh_access"),
    (r'crontab', "persistence_cron"),
    (r'\.(bashrc|zshrc|profile)', "shell_rc_mod"),
]

def _scan_memory_content(content: str) -> Optional[str]:
    """扫描记忆内容，发现威胁返回错误字符串"""
    # 检测不可见 Unicode
    for char in _INVISIBLE_CHARS:
        if char in content:
            return f"Blocked: 不可见 Unicode 字符 U+{ord(char):04X}"
    
    # 检测威胁模式
    for pattern, pid in _MEMORY_THREAT_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            return f"Blocked: 匹配威胁模式 '{pid}'"
    
    return None  # 安全
```

## 第 4 层：上下文文件注入扫描

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'act\s+as\s+(if|though)\s+.*no\s+(restrictions|limits)', "bypass_restrictions"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials)', "read_secrets"),
    (r'<!--[^>]*(?:ignore|override|system|secret|hidden)[^>]*-->', "html_comment_injection"),
    (r'<\s*div\s+style\s*=\s*["\'].*display\s*:\s*none', "hidden_div"),
    (r'base64\s+(-d|--decode)\s*\|', "base64_decode_pipe"),
]

def _scan_context_content(content: str, filename: str) -> str:
    """扫描上下文文件（SOUL.md, AGENTS.md 等）"""
    findings = []
    
    # 检测不可见 Unicode
    for char in _CONTEXT_INVISIBLE_CHARS:
        if char in content:
            findings.append(f"invisible unicode U+{ord(char):04X}")
    
    # 检测威胁模式
    for pattern, pid in _CONTEXT_THREAT_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            findings.append(pid)
    
    if findings:
        logger.warning("Context file %s blocked: %s", filename, ", ".join(findings))
        return f"[BLOCKED: {filename} contained potential prompt injection ({', '.join(findings)}). Content not loaded.]"
    
    return content  # 安全，返回原始内容
```

## 第 5 层：终端命令启发式检测

```python
_DESTRUCTIVE_PATTERNS = re.compile(
    r"""(?:^|\s|&&|\|\||;|`)(?:
        rm\s|rmdir\s|
        mv\s|
        sed\s+-i|
        truncate\s|
        dd\s|
        shred\s|
        git\s+(?:reset|clean|checkout)\s
    )""",
    re.VERBOSE,
)

_REDIRECT_OVERWRITE = re.compile(r'[^>]>[^>]|^>[^>]')

def _is_destructive_command(cmd: str) -> bool:
    """启发式：此终端命令是否像修改/删除文件？"""
    if not cmd:
        return False
    if _DESTRUCTIVE_PATTERNS.search(cmd):
        return True
    if _REDIRECT_OVERWRITE.search(cmd):
        return True
    return False
```

## 安全扫描执行时机

| 时机 | 扫描内容 | 扫描器 |
|------|----------|--------|
| 技能创建 | 整个技能目录 | Skills Guard |
| 技能编辑/补丁 | 整个技能目录 | Skills Guard |
| 记忆写入 | 条目内容 | Memory Scanner |
| 上下文文件加载 | SOUL.md, AGENTS.md 等 | Context Scanner |
| 技能安装（Hub） | 整个技能目录 | Skills Guard |

## 回滚机制

```python
# 技能创建/编辑后扫描
scan_error = _security_scan_skill(skill_dir)
if scan_error:
    # 自动回滚到修改前的状态
    _atomic_write_text(target, original_content)
    return {"success": False, "error": scan_error}
```

## 与其他 Agent 框架对比

| 特性 | Hermes | Cursor | Claude Desktop |
|------|--------|--------|----------------|
| 技能安全扫描 | ✅ 100+ 模式 | N/A | N/A |
| 信任级别策略 | ✅ 4 级 | N/A | N/A |
| 记忆内容扫描 | ✅ | N/A | N/A |
| 上下文文件扫描 | ✅ | N/A | N/A |
| Unicode 注入检测 | ✅ 17 字符 | ❌ | ❌ |
| 自动回滚 | ✅ | N/A | N/A |
| 破坏性命令检测 | ✅ 启发式 | ❌ | ❌ |

## 危险命令审批系统（tools/approval.py — 877 行）

当 agent 执行的终端命令匹配危险模式时，系统拦截并要求用户确认。

### 三种审批模式

```yaml
# config.yaml
approvals:
  mode: smart   # manual | smart | off
```

| 模式 | 行为 |
|------|------|
| `manual` | 所有匹配危险模式的命令都要人工确认 |
| `smart` | 先用 auxiliary LLM 评估风险，低风险自动放行，高风险才问用户 |
| `off`（yolo） | 跳过所有审批（危险，仅限可信环境） |

### 审批选项（CLI 交互）

用户看到危险命令后可选择：
- **once** — 本次允许
- **session** — 本次会话内同类命令都允许
- **always** — 永久允许（写入 config.yaml）
- **deny** — 拒绝执行

超时未响应（45 秒）→ 默认拒绝（fail-closed）。

### 危险模式检测

匹配规则涵盖：
- 破坏性操作：`rm -rf`、`mkfs`、`dd`、`truncate` 等
- 权限提升：`sudo`、`su`、`chmod 777`
- 敏感文件写入：`/etc/`、`~/.ssh/`、`~/.hermes/.env`
- 网络操作：`curl | bash`、端口监听
- 环境变量操控：覆盖 `PATH`、`LD_PRELOAD`

### Per-session 状态

审批状态按 session 隔离（`contextvars.ContextVar`），gateway 多用户并发时互不影响。"session" 级别的允许只在当前会话有效，不跨 session。

## 供应链咨询检查器（hermes_cli/security_advisories.py，2026-05-12）

针对 PyPI 上单包被投毒的攻击（如 2026-05-12 命中 `mistralai==2.4.6` 的
Mini Shai-Hulud 蠕虫），Hermes 新增了运行时**供应链咨询检查器**，为已经
中招的用户提供检测与修复指引。

### ADVISORIES 目录

```python
@dataclass(frozen=True)
class Advisory:
    advisory_id: str          # 如 "shai-hulud-2026-05"
    package: str              # 如 "mistralai"
    bad_versions: ...         # 受影响版本（如 "2.4.6"）
    # 标题、说明、修复步骤等

ADVISORIES: tuple[Advisory, ...] = (...)   # 目前 1 条
```

新增一条咨询只需添加一个 `Advisory` dataclass 条目。

### 检测与提示流程

- `detect_compromised()` 用 `importlib.metadata.version()` 检查本机已装版本
  —— 不依赖 pip，可在缺少 pip 的 uv venv 中工作。
- Banner 缓存（`~/.hermes/cache/advisory_banner_seen`）将启动横幅限制为
  每条咨询每 24 小时一次。
- 用户确认（ack）持久化到 `config.yaml` 的 `security.acked_advisories`，
  确认后不再重复提示。
- 接入点：`hermes doctor`（首先运行，打印完整修复块）、
  `hermes doctor --ack <id>`（消除某条咨询）、`cli.py` 交互/单查询分支
  （stderr 短横幅指向 `hermes doctor`）、`gateway/run.py` 启动
  （在 `gateway.log` 中输出运维可见的警告）。

## 懒安装框架与分层安装回退（tools/lazy_deps.py）

为减小基础安装体积并降低供应链暴露面，opt-in 后端改为**首次使用时按需
安装**，而非安装时全量拉取。

- `LAZY_DEPS` 白名单将带命名空间的功能键（如 `tts.elevenlabs`、
  `memory.honcho`、`provider.bedrock`）映射到 pip spec。
- `ensure(feature)` 通过 `uv → pip → ensurepip` 阶梯在当前 venv 中安装
  缺失依赖。
- 严格的 spec 安全正则会拒绝 URL、文件路径、shell 元字符、pip 标志注入、
  控制字符 —— 只接受按名称引用的 PyPI 包。
- 受 `security.allow_lazy_installs` 开关控制（默认 true）。
- **分层安装回退**：一个被隔离/撤回的 PyPI 包不再静默把全新安装降级为
  "仅核心"，安装器会保留其它所有 extra 并告知用户最终落到了哪一层。

配套的依赖锁定策略（commit `04b1fda`）：为 5 个未锁定的宽松依赖添加了
版本上界，并在文档中记录了供应链策略（精确 pin + `uv.lock` + 哈希校验
安装路径 + CI 的 `uv lock --check` 漂移门禁）。

## 额外安全层

- `tools/tirith_security.py` — Tirith 安全策略引擎（homograph URL、pipe-to-shell、terminal 注入）
- `tools/url_safety.py` — URL 安全检查（SSRF 防护：拦截私有网络、云元数据地址、验证重定向）
- `tools/osv_check.py` — 依赖恶意软件扫描（OSV 数据库）

## YOLO 模式可见性（2026-05-15）

`--yolo` 模式会绕过所有危险命令审批。为避免用户忘记自己处于此状态，
CLI 现在显式展示该状态（commit `b6e0741`）：

- **Banner**：仅在 YOLO 激活时以红色显示
  `⚠ YOLO mode — all approval prompts bypassed` 一行；默认情况静默。
- **状态栏**：在三种宽度档（<52、<76、≥76）的纯文本回退与 fragments
  构建器中都追加红色 `⚠ YOLO` 片段。

## 相关页面

- [[memory-system-architecture]] — 记忆内容安全扫描机制
- [[skills-system-architecture]] — 技能安装时的安全扫描与信任策略
- [[prompt-builder-architecture]] — 上下文文件注入扫描防护

## 相关文件

- `tools/skills_guard.py` — Skills Guard 安全扫描
- `tools/memory_tool.py` — Memory 内容扫描
- `agent/prompt_builder.py` — 上下文文件扫描
- `run_agent.py` — 终端命令启发式检测
- `tools/approval.py` — 命令审批（31 模式）
- `tools/tirith_security.py` — Tirith 安全策略
- `tools/url_safety.py` — SSRF 防护
- `tools/osv_check.py` — 恶意软件扫描
- `hermes_cli/security_advisories.py` — 供应链咨询检查器（451 行）
- `tools/lazy_deps.py` — 懒安装框架与白名单（608 行）
- `hermes_cli/banner.py` — YOLO 模式横幅警告
