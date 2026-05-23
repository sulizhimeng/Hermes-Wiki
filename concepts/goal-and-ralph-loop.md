---
title: /goal 与 /subgoal（Ralph Loop）
created: 2026-05-22
type: concept
tags: [goal, ralph-loop, slash-command, persistence, judge, autonomous]
sources: [hermes_cli/goals.py, hermes_cli/config.py, cli.py, gateway/run.py, tui_gateway/server.py]
---

# `/goal` 与 `/subgoal`（Ralph Loop）

## 概述

`/goal` 把"持续工作直到目标完成"做成一等公民。用户设定一个标准目标后，每回合结束都由辅助 LLM **判官** 评判是否达成：

- **达成** → 停下来告诉用户
- **未达成** → 自动把"继续推进目标"作为下一回合 user message 注入

这是 **Ralph loop** 模式的官方实现，在 Hermes v0.13 (`#18262/#18275`) 引入，v0.14 (`#25449`) 补全 `/subgoal`。

> 关键不变量：续推 prompt 是普通 user message，不动 system prompt → **保护 prompt cache**。

源码：`hermes_cli/goals.py`（762 行）+ CLI/Gateway/TUI 三处钩子。

---

## 用户接口

### `/goal`

| 子命令 | 行为 |
|--------|------|
| `/goal <text>` | 设新目标，立即用目标文本作首轮 user message 注入 |
| `/goal status` | 显示当前目标状态（active/paused/done）、turns used、subgoals |
| `/goal pause` | 手动暂停（可 resume） |
| `/goal resume` | 解暂停，可选重置 turn budget |
| `/goal clear` / `stop` / `done` | 清除目标 |

### `/subgoal`

| 子命令 | 行为 |
|--------|------|
| `/subgoal` | 列出当前 subgoals |
| `/subgoal <text>` | 追加一条判定标准 |
| `/subgoal remove <n>` | 删第 n 条（1-based） |
| `/subgoal clear` | 清空 |

Subgoals 是**约束**：每条都要在 agent 的响应中有具体证据，才算完成。判官明确要求"不接受 'all requirements met' 这种空洞话"。

---

## 核心数据：`GoalState`

`hermes_cli/goals.py:142-161`

```python
@dataclass
class GoalState:
    goal: str                              # 目标文本
    status: str                            # active / paused / done / cleared
    turns_used: int                        # 已用回合
    max_turns: int                         # 预算（默认 20）
    created_at: float
    last_turn_at: float
    last_verdict: str                      # done / continue / skipped
    last_reason: str                       # 判官给的理由
    paused_reason: str                     # 自动暂停原因
    consecutive_parse_failures: int        # 判官输出解析失败计数
    subgoals: List[str]                    # 用户中途补的标准
```

序列化：`to_json()` / `from_json()` 在 `goals.py:163-185`，带向后兼容反序列化。

---

## 持久化

存在 SessionDB `state_meta` 表，key `goal:<session_id>`：

```python
save_goal(session_id, state)   # goals.py:260-270
load_goal(session_id)          # goals.py:239-257
clear_goal(session_id)         # goals.py:273-279
```

**保存时机**：每次判官调用后（`goals.py:727`）+ 任何状态变更（设置/暂停/完成/清除/加 subgoal/删 subgoal）。

SessionDB 连接按 `hermes_home` path 缓存（`goals.py:206-236`），保证 profile 切换正确。

---

## 判官（Judge）

`judge_goal(goal, last_response, timeout=30, subgoals=None) → (verdict, reason, parse_failed)`
位置：`goals.py:371-463`

### 流程

1. **空校验**（`goals.py:398-402`）：goal 或 response 为空 → 返回 `"continue"`，跳过判官调用
2. **客户端解析**（`goals.py:410-417`）：`auxiliary_client.get_text_auxiliary_client("goal_judge")` 路由判官模型
   - 无可用客户端 → 返回 `"continue"`（**fail-open**）
3. **Prompt 选择**（`goals.py:419-437`）
   - 有 subgoals → `JUDGE_USER_PROMPT_WITH_SUBGOALS_TEMPLATE`（goals.py:120-134）
   - 否则 → `JUDGE_USER_PROMPT_TEMPLATE`（goals.py:111-116）
4. **API 调用**（`goals.py:439-450`）：temperature=0，system prompt `JUDGE_SYSTEM_PROMPT`（goals.py:95-108），max_tokens 默认 4096，timeout 30s
5. **解析**（`_parse_judge_response`，`goals.py:322-368`）：单行 JSON `{"done": <bool>, "reason": "<text>"}`
   - 容忍 markdown fence
   - 容忍前置散文
   - `"true"/"yes"/"1"/"done"` 字符串自动转 bool
   - **解析失败 → `parse_failed=True`**，verdict=continue（fail-open）

### Judge System Prompt（节选）

```
A goal is DONE only when:
- The response explicitly confirms the goal was completed, OR
- The response clearly shows the final deliverable was produced, OR
- The response explains the goal is unachievable / blocked / needs user input
  (treat this as DONE with reason describing the block).

Otherwise the goal is NOT done — CONTINUE.

Reply ONLY with a single JSON object on one line:
{"done": <true|false>, "reason": "<one-sentence rationale>"}
```

### Judge Prompt（带 subgoals 时的关键句）

> *For each numbered criterion above, find concrete evidence in the agent's response that the criterion is satisfied. Do not accept generic phrases like 'all requirements met' or 'implying it was done' — require specific evidence (a file contents excerpt, an output line, a command result). If ANY criterion lacks specific evidence in the response, the goal is NOT done — return CONTINUE.*

---

## 续推 Prompt 模板

### 无 subgoals（`goals.py:71-77`）
```
[Continuing toward your standing goal]
Goal: {goal}

Continue working toward this goal. Take the next concrete step. If you believe
the goal is complete, state so explicitly and stop. If you are blocked and
need input from the user, say so clearly and stop.
```

### 有 subgoals（`goals.py:82-92`）

包含编号 subgoal 列表 + 必须全满足的措辞。

**这些都是普通 user message**，作为下一回合输入注入。不动 system prompt = 不毁 prompt cache。

---

## 自动暂停

### 预算耗尽（`goals.py:711-725`）

`turns_used >= max_turns` 时：
- status → `paused`
- paused_reason → `"turn budget exhausted (n/20)"`
- 提示用户 `/goal resume` 或 `/goal clear`

### 判官连续解析失败（`goals.py:687-709`）

`consecutive_parse_failures` 达 `DEFAULT_MAX_CONSECUTIVE_PARSE_FAILURES`（默认 3）时：
- status → `paused`
- paused_reason → `"judge model returned unparseable output N turns in a row"`
- 提示用户给 `auxiliary.goal_judge` 配更严谨的模型

### 用户 Ctrl+C 中断（`cli.py:8944-8953`）

回合被 Ctrl+C 打断 → 不判，直接暂停。

---

## 三处钩子

`/goal` + `/subgoal` 在 CLI、Gateway、TUI 都注册，语义一致：

| Surface | 钩子位置 |
|---------|----------|
| CLI | `cli.py:8152-8154`（dispatch）+ `_handle_goal_command` (8739-8804) + `_handle_subgoal_command` (8805-8879) + `_maybe_continue_goal_after_turn` (8880-8995) |
| Gateway | `gateway/run.py:10608` `_post_turn_goal_continuation()` |
| TUI Gateway | `tui_gateway/server.py:3477` |

### CLI 续推流程（`_maybe_continue_goal_after_turn`）

1. 检查 goal active（line 8901）
2. 检查抢占：若已有真实 user 消息排队 → 延后判官（line 8914-8935）
3. 检查中断：Ctrl+C → 自动暂停（line 8944-8953）
4. 提取最后 assistant 响应（line 8955-8974）
5. 跳过空响应（line 8980）
6. `mgr.evaluate_after_turn(last_response, user_initiated=True)`（line 8983）
7. 打印状态消息
8. 若判 continue → 把续推 prompt 入 `_pending_input` 队列

Gateway 用 `MessageEvent` FIFO 队列实现相同抢占语义。

---

## 配置

### config.yaml

```yaml
goals:
  max_turns: 20    # 默认每次 /goal 的回合预算

auxiliary:
  goal_judge:
    provider: openrouter        # 推荐严谨 JSON 输出的模型
    model: google/gemini-3-flash-preview
    max_tokens: 4096            # 推理模型需要（隐藏 token 在前），默认 4096
```

`max_tokens` 默认 4096 而不是 200：DeepSeek-V4 / QwQ 等推理模型会先 emit 大量隐藏 token 再写 JSON，200 token 会被截断。

### 常量（`goals.py:47-68`）

| 常量 | 值 | 说明 |
|------|-----|------|
| `DEFAULT_MAX_TURNS` | 20 | 默认回合预算 |
| `DEFAULT_JUDGE_TIMEOUT` | 30.0 | 判官 API 超时（秒） |
| `DEFAULT_JUDGE_MAX_TOKENS` | 4096 | 判官输出 token 预算 |
| `_JUDGE_RESPONSE_SNIPPET_CHARS` | 4000 | 送给判官的 response 字符上限 |
| `DEFAULT_MAX_CONSECUTIVE_PARSE_FAILURES` | 3 | 暂停阈值 |

---

## 与 conversation_loop 的关系

**没有直接依赖**。`conversation_loop.py` 不 import `goals`。续推完全是 **post-turn**：

```
run_conversation(agent, user_msg, ...) → returns {final_response, ...}
        │
        ▼
  caller 抽 final_response
        │
        ▼
  mgr.evaluate_after_turn(final_response) → {should_continue, status_msg}
        │
        ▼
  若 should_continue → 把续推 prompt 当作新 user_msg 入队
        │
        ▼
  下一次 run_conversation() 自动用续推 prompt 跑
```

这种**调用者驱动**的设计：

1. 不需要改 agent loop
2. 不污染 system prompt → prompt cache 完整
3. 用户实时输入自然抢占（real user message 排在续推前）

---

## 设计原则

1. **Fail-open**：判官出错（API/parse） → 默认 continue，从不假阳性 done。Turn budget 是兜底
2. **不改 system prompt**：续推是 user message，保 cache
3. **用户抢占**：真实 user message 自动挤掉续推
4. **CLI / Gateway / TUI 完全对齐**：同一套语义
5. **无状态判官**：每回合独立调用，不带历史，temperature=0
6. **Subgoals 是约束不是支线**：必须全有证据，否则继续

---

## 文件索引

| 功能 | 文件 : 行 |
|------|-----------|
| 设计动机文档串 | `goals.py:1-28` |
| 常量 | `goals.py:47-68` |
| Prompt 模板 | `goals.py:71-134` |
| Judge system prompt | `goals.py:95-108` |
| GoalState dataclass | `goals.py:142-161` |
| 持久化函数 | `goals.py:239-280` |
| 判官响应解析 | `goals.py:322-368` |
| `judge_goal()` | `goals.py:371-463` |
| `GoalManager` 类 | `goals.py:471-748` |
| `evaluate_after_turn` | `goals.py:620-737` |
| `add_subgoal` | `goals.py:573-586` |
| `remove_subgoal` | `goals.py:588-599` |
| `render_subgoals_block` | `goals.py:189-194` |
| config schema | `hermes_cli/config.py:1260-1266` |
| CLI dispatch | `cli.py:8152-8154` |
| `_handle_goal_command` | `cli.py:8739-8804` |
| `_handle_subgoal_command` | `cli.py:8805-8879` |
| `_get_goal_manager` | `cli.py:8706-8737` |
| Post-turn CLI hook | `cli.py:8880-8995` |
| Post-turn Gateway hook | `gateway/run.py:_post_turn_goal_continuation` @ 10608 |
| Gateway GoalManager binder | `gateway/run.py` @ 10397, 10416 |
| Post-turn TUI hook | `tui_gateway/server.py` @ 3477 |
| TUI GoalManager binder | `tui_gateway/server.py` @ 4862, 4875 |
| Tests | `tests/hermes_cli/test_goals.py` |

---

*Last verified: 2026-05-22, HEAD `09afafb87`*
