---
title: CLI 架构与终端交互设计
created: 2026-04-07
updated: 2026-05-20
type: concept
tags: [architecture, cli, terminal, ux, tui, ink]
sources: [cli.py, hermes_cli/, ui-tui/, tui_gateway/]
---

# CLI 架构与终端交互设计

## 设计原理

Hermes 实际上有 **两个完全独立的 CLI 前端**（共用同一后端 agent）：

1. **经典 CLI**（`cli.py` + `hermes_cli/`） —— Python `prompt_toolkit` + `rich`，进 `hermes` 默认入口。
2. **Ink TUI**（`ui-tui/` + `tui_gateway/`） —— v0.11.0 起的 React/Ink 重写，进 `hermes --tui`。Python 这边跑一个 JSON-RPC server（`tui_gateway/`），Ink Node 进程通过 RPC 调 agent。

> Ink TUI 是 v0.11.0 ~310 commits 的重写（@OutThisLife + Teknium）。v0.12.0 完成 ~57% 冷启动加速，v0.14.0 进一步 −19 秒。

## 经典 CLI

基于 `prompt_toolkit` 和 `rich` 构建。

## 核心组件

```python
# cli.py
class HermesCLI:
    """Hermes CLI 主类"""
    
    def __init__(self):
        self.agent = None
        self.config = load_cli_config()
        self.session_db = SessionDB(...)
        self.todo_store = TodoStore()
    
    def run(self):
        """主循环"""
        while True:
            user_input = self._get_input()  # prompt_toolkit 输入
            if user_input.startswith("/"):
                self._handle_command(user_input)
            else:
                self._handle_message(user_input)
```

## 输入系统

```python
from prompt_toolkit import PromptSession
from prompt_toolkit.history import FileHistory
from prompt_toolkit.auto_suggest import AutoSuggestFromHistory

session = PromptSession(
    history=FileHistory("~/.hermes/input_history"),
    auto_suggest=AutoSuggestFromHistory(),
    completer=SlashCommandCompleter(),  # 定义在 hermes_cli/commands.py
)

user_input = session.prompt(get_active_prompt_symbol())  # 提示符号通过 skin engine 配置
```

### 斜杠命令补全

```python
class SlashCommandCompleter(Completer):
    def get_completions(self, document, complete_event):
        text = document.text_before_cursor
        if text.startswith("/"):
            for cmd_name, cmd_def in COMMANDS.items():
                if cmd_name.startswith(text[1:]):
                    yield Completion(cmd_name, start_position=-len(text[1:]))
```

## 显示系统

### KawaiiSpinner

```python
# agent/display.py
class KawaiiSpinner:
    """动画加载指示器"""
    
    SPINNERS: dict          # 9 种命名动画集 ('dots', 'bounce', 'grow', ...)
    KAWAII_WAITING: list     # 10 个多字符颜文字
    KAWAII_THINKING: list    # 15 个多字符颜文字
    THINKING_VERBS: list    # 15 个动词 ("pondering", "contemplating", "musing", "cogitating", "ruminating", ...)
    
    def show(self, message: str):
        """显示加载动画"""
        # 使用 Rich 面板和动画
```

### 工具调用预览

```python
def build_tool_preview(tool_name: str, args: dict) -> str:
    """构建工具调用预览"""
    preview = f"🔧 {tool_name}("
    for key, value in list(args.items())[:3]:
        preview += f"\n  {key}={preview_value(value)},"
    preview += "\n)"
    return preview

def get_cute_tool_message(tool_name: str) -> str:
    """获取可爱的工具执行消息"""
    emoji = _get_tool_emoji(tool_name)
    return f"{emoji} Calling {tool_name}..."
```

## Skin 引擎

```python
# hermes_cli/skin_engine.py
@dataclass
class SkinConfig:
    """皮肤配置数据类"""
    ...

# 模块级函数（非类）
def init_skin_from_config(): ...
def get_active_skin() -> SkinConfig: ...
def list_skins() -> list: ...
def set_active_skin(name: str): ...

# 配置示例
# ~/.hermes/config.yaml
display:
  skin: "default"  # 或自定义皮肤名称
```

## 优越性分析

### 与其他 Agent 框架对比

| 特性 | Hermes | Claude Code | Codex CLI |
|------|--------|-------------|-----------|
| 斜杠命令补全 | ✅ 自动 | ✅ | ❌ |
| 多行编辑 | ✅ | ✅ | ✅ |
| 输入历史 | ✅ 文件持久化 | ✅ | ✅ |
| 动画加载 | ✅ KawaiiSpinner | ✅ 简单 | ✅ 简单 |
| 主题系统 | ✅ Skin Engine | ❌ | ❌ |
| 工具调用预览 | ✅ 格式化 | ✅ | ❌ |

## v0.11.0+ 新增子命令 / 斜杠命令

### 子命令（`hermes <subcommand>`）

| 命令 | 源码 | 版本 |
|------|------|------|
| `hermes proxy` | `hermes_cli/proxy/` | v0.14.0 — OAuth → OpenAI 本地代理（详见 [[hermes-proxy]]） |
| `hermes kanban {add,show,list,take,complete,block,unblock,comment,link,decompose,specify,diagnostics,swarm,serve,boards}` | `hermes_cli/kanban*.py` | v0.13.0（详见 [[multi-agent-kanban]]） |
| `hermes curator {status,run,pause,resume,pin,unpin,restore,list-archived,archive,prune}` | `hermes_cli/curator.py` + `agent/curator.py` | v0.12.0+ |
| `hermes -z <prompt>` | `hermes_cli/oneshot.py` | v0.12.0 — 非交互一次性 |
| `hermes update --check` | `hermes_cli/main.py` | v0.12.0 — 升级前 preflight + opt-in HERMES_HOME 备份 |
| `hermes migrate xai [--apply] [--no-backup]` | `hermes_cli/migrate.py` + `hermes_cli/xai_retirement.py` | post-v0.14.0 — xAI 退役模型批量改 config |
| `hermes acp --setup-browser` | `hermes_cli/main.py` | v0.14.0 — Zed ACP Registry 安装路径 bootstrap |
| `hermes proxy --provider nous\|xai` | 同上 | v0.14.0 |

### 斜杠命令（session 内）

| 命令 | 版本 | 不变量 |
|------|------|--------|
| `/goal <text>` | v0.13.0 | Ralph 循环，prompt cache 不失效（详见 [[goal-loop-and-steering]]） |
| `/steer <text>` | v0.11.0 强化 | 当回合补丁，next-turn note 注入 |
| `/queue <text>` | v0.13.0 ACP | FIFO 等当前 turn 完成 |
| `/handoff <target>` | v0.14.0 | 现场迁移整个 session 到目标 model/persona/profile |
| `/reload-skills` | v0.11.0 | 重扫 `~/.hermes/skills/` 不失效 prompt cache |
| `/reload-mcp` | v0.11.0 | 失效 prompt cache，弹确认 |
| `/clear` (带确认) | v0.11.0 | — |
| `/mouse` | v0.12.0 | 关 ConPTY 幻影鼠标注入（@kevin-ho） |
| `/kanban ...` | v0.13.0 | 同 `hermes kanban` argparse 表面 |
| `/board` | v0.13.0 | 看板 dashboard 入口 |
| `/curator status` | v0.12.0 | — |

`/steer` 和 `/queue` 也由 ACP 客户端（Zed / VS Code / JetBrains，@HenkDz）触发。

## TUI 增强（Ink TUI）

| 功能 | 版本 |
|------|------|
| Sticky composer + OSC-52 剪贴板 + 稳定 picker keys | v0.11.0 |
| 状态栏 per-turn stopwatch + git branch | v0.11.0 |
| 子代理 spawn observability overlay | v0.11.0 |
| LaTeX 渲染（@austinpickett） | v0.12.0 |
| `d` 删 session in `/resume` picker | v0.12.0 |
| `/reload` 热重载 `.env` | v0.12.0 |
| 可插拔 busy-indicator（@OutThisLife） | v0.12.0 |
| 自动 resume 上次 session（opt-in） | v0.12.0 |
| 扩展明亮终端自动检测 | v0.12.0 |
| 修饰键鼠标滚轮行滚动 | v0.12.0 |
| `/model` picker inline 鉴权（@austinpickett） | v0.13.0 |
| 启动 banner 可折叠区块（@kshitijk4poor） | v0.13.0 |
| 状态栏 context-compression counter | v0.13.0 |
| 点击 OSC8 hyperlink（@OutThisLife） | v0.14.0 |

## 相关页面

- [[configuration-and-profiles]] — 配置管理与 Profile 系统
- [[hook-system-architecture]] — Hook 与插件扩展系统
- [[session-search-and-sessiondb]] — 会话搜索与 SessionDB
- [[voice-mode-architecture]] — 语音模式（Push-to-talk → STT → TTS）
- [[skin-engine]] — 皮肤/主题自定义
- [[context-references]] — @file/@diff/@url 引用系统
- [[worktree-isolation]] — Git Worktree 并行隔离
- [[code-execution-sandbox]] — 代码执行沙箱
- [[multi-agent-kanban]] — `/kanban` 与 `hermes kanban`
- [[goal-loop-and-steering]] — `/goal` `/steer` `/queue` `/handoff`
- [[hermes-proxy]] — `hermes proxy`

## 相关文件

- `cli.py` — 经典 CLI 主类（660KB；大单体文件）
- `hermes_cli/main.py` — 入口点和子命令
- `hermes_cli/commands.py` — 斜杠命令定义
- `hermes_cli/dump.py` — `hermes dump` 环境摘要（纯文本，用于调试/提 issue）
- `agent/display.py` — 显示系统
- `hermes_cli/skin_engine.py` — 皮肤引擎
- `hermes_cli/goals.py` — `/goal` Ralph 循环
- `hermes_cli/kanban*.py` — 看板 CLI 族
- `hermes_cli/curator.py` — `hermes curator` 子命令
- `hermes_cli/proxy/` — `hermes proxy` server
- `ui-tui/` — Ink TUI（Node.js / React）
- `tui_gateway/` — Python JSON-RPC backend for Ink TUI
