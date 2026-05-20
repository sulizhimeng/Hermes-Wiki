---
title: Goal Loop / Steer / Queue / Handoff —— Session 级控制原语
created: 2026-05-20
updated: 2026-05-20
type: concept
tags: [session-control, steer, goal, handoff, ralph-loop]
source_files:
  - hermes_cli/goals.py
  - run_agent.py
  - hermes_cli/main.py
verified_against: hermes-agent HEAD (2026-05-20)
---

# Session 级控制原语：`/goal` / `/steer` / `/queue` / `/handoff`

Hermes v0.11.0 / v0.13.0 / v0.14.0 沿着同一条线索 ——「**不打断 turn，不污染 prompt cache，从外部操纵正在跑的 agent**」—— 加了 4 个 session 级控制原语。

| 原语 | 触发 | 时机 | 修改 prompt cache |
|------|------|------|------------------|
| `/goal <text>` | 用户 | 每回合结束 judge | **不修改** —— continuation 是普通 user message |
| `/steer <text>` | 用户 / ACP 客户端 | 当前 turn 的下一次 tool call 之后 | **不修改** —— 注入 next-turn note |
| `/queue <text>` | 用户 / ACP 客户端 | 当前 turn 结束后 | **不修改** —— FIFO 投递 |
| `/handoff <target>` | 用户 | 立即（同步迁移） | 因目标模型 cache 不同而冷启 |

---

## 1. `/goal` —— Ralph 循环

源码：`hermes_cli/goals.py`，762 行。

### 状态

```python
@dataclass
class Goal:
    text: str                     # 用户给的目标描述
    session_id: str
    turns_used: int = 0
    max_turns: int = 20           # DEFAULT_MAX_TURNS
    paused: bool = False
    judge_timeout: float = 30.0   # DEFAULT_JUDGE_TIMEOUT
    judge_output_budget: int = ...# reasoning 模型给宽
    created_at: str
    last_judged_at: Optional[str]
```

持久化在 SessionDB 的 `state_meta` 表，键 `goal:<session_id>` —— **`/resume` 自动接上**未完成 goal。

### 工作流

```
user issues "/goal <text>"
   │
   ▼
turn N completes
   │
   ▼
judge call (auxiliary model)
   ├─ verdict="done"      → goal cleared, 通知用户
   ├─ verdict="continue"  → 把 continuation prompt 当 user message 喂回 run_conversation
   ├─ verdict="failed"    → fail-OPEN → 等同 continue（不卡住进展）
   └─ judge timeout       → 等同 continue
   │
   ▼
turn N+1 starts (turns_used += 1)
   ⋮
   ▼ 触发停止的任意条件：
      - judge 说 done
      - turns_used >= max_turns
      - 用户 /goal --pause 或 /goal --clear
      - 用户发新消息（抢占当回，re-judge）
```

### 关键不变量（源码注释直引）

> - The continuation prompt is just a normal user message appended to the session via `run_conversation`. **No system-prompt mutation, no toolset swap — prompt caching stays intact.**
> - Judge failures are **fail-OPEN**: `continue`. A broken judge must not wedge progress; the turn budget is the backstop.
> - When a real user message arrives mid-loop it **preempts** the continuation prompt and also pauses the goal loop for that turn (we still re-judge after).

> 这意味着 `/goal` 不是"长 running task"工具 —— 它就是"自动追问"。需要分布式 / 跨 session / 重试的，用 [[multi-agent-kanban]]。

### CLI

```
/goal <text>            # 设目标
/goal --status          # 显示当前 goal + 已用轮次 + 剩余预算
/goal --pause           # 暂停（不取消）
/goal --resume          # 恢复
/goal --clear           # 取消
/goal --max-turns N     # 调预算
```

`hermes_cli/main.py` 把 `GoalManager` 单例同时注入到 CLI 路径和 gateway 路径，**两个入口共用同一份代码**。

---

## 2. `/steer` —— 中途插话

引入：v0.11.0 (#12116)。强化：v0.13.0 ACP 加入。

机制：

- 用户发 `/steer <prompt>`。
- 不打断当前 turn，不发起新 turn。
- agent 跑到下一次 tool call 完成的 hook，看到一段 `[STEER NOTE] <prompt>` 注入文本。
- 这段文本以**轮内补丁**的形式插入，**不写入 cache 的稳定后缀**。

效果：你能在 agent 跑了一半的时候说"先去看 schema 文件再继续"，而它会在下一次 tool call 后看到你的提示，把它纳入决策，但本回合的预算 / cache / 状态都不变。

ACP 集成（v0.13.0）：Zed / VS Code / JetBrains 客户端可以直接发 `/steer`，由 ACP runtime 桥接 —— 用户在 IDE 里点按钮就能 steer。

---

## 3. `/queue` —— FIFO 排队后续消息

引入：v0.13.0 ACP（@HenkDz）。

机制：

- 用户在 agent 跑的时候发 `/queue <msg>`。
- 消息排入 FIFO，**不打断 turn**。
- 当前 turn 完结时，逐条按顺序投递。

vs `/steer`：

- `/steer` 是**当回打补丁**，agent 同一 turn 内消化。
- `/queue` 是**下一回合 user message**，每条触发新 turn。

---

## 4. `/handoff` —— 现场迁移 session

引入：v0.14.0。

```
/handoff --model claude-opus-4-7
/handoff --provider anthropic
/handoff --profile coder
/handoff --persona translator
```

机制：

- 保留全部 messages / tool calls / context references / memory state。
- 切换 agent 实例 = 新 provider client / 新 transport / 新 system prompt（按 profile / persona 重算）。
- prompt cache 必然冷（目标 cache 可能根本没记录过这个 prefix），但**对话连续性**保留。

vs `/clear`：

- `/clear` 是"清记忆"。
- `/handoff` 是"换处理器，记忆全留"。

vs 多 Profile（[[configuration-and-profiles]]）：

- 多 profile 是**完全隔离的 agent 实例**，session 不共享。
- `/handoff` 是"同一 session 换 agent"。

---

## 5. 共同设计哲学

四个原语共享一句话：

> "**不要为了控制 agent 而打断 agent**。"

具体实现策略：

| 策略 | 体现 |
|------|------|
| 不动 system prompt | `/goal` / `/steer` / `/queue` 全都只在 user message 流里加内容 |
| 不动 toolset | 同上 |
| Fail-OPEN | judge 错了就 continue，绝不 wedge |
| 用户消息抢占 | 任何用户 input 都优先于自动机 |
| 持久化在 state_meta | `/resume` 不丢 goal |
| ACP 桥接 | IDE / 编辑器作为一等触发器，不只是命令行 |

---

## 6. 与现有机制的关系

| 需求 | 用哪个 |
|------|--------|
| 单回合内打补丁 | `/steer` |
| 排队后续指令 | `/queue` |
| 持续追问直到完成 | `/goal` |
| 换模型/profile 不丢上下文 | `/handoff` |
| **跨 session 持久化任务** | **Kanban** [[multi-agent-kanban]] |
| **fork 并行 worker** | `delegate_task` [[multi-agent-architecture]] |
| **跨 Profile 通信** | gateway 平台 + Profile [[configuration-and-profiles]] |

---

## 7. 源码索引

```
hermes_cli/goals.py:1             模块 docstring (Ralph 循环说明)
hermes_cli/goals.py:48            DEFAULT_MAX_TURNS = 20
hermes_cli/goals.py:50            DEFAULT_JUDGE_TIMEOUT = 30.0
hermes_cli/goals.py:13-25         不变量列表（continuation = user msg / fail-OPEN / 抢占）
hermes_cli/main.py                GoalManager 接入点（CLI + gateway 双路径）
run_agent.py                      run_conversation 的 continuation prompt 入口
agent/conversation_loop.py        steer note 注入 hook
```
