---
title: Kanban 编排系统
created: 2026-05-19
updated: 2026-05-19
type: concept
tags: [architecture, agent, delegation, automation, multi-platform]
sources: [hermes_cli/kanban_db.py, hermes_cli/kanban.py, tools/kanban_tools.py, hermes_cli/kanban_decompose.py, hermes_cli/kanban_diagnostics.py, gateway/run.py]
---

# Kanban 编排系统

## 概述

Kanban 是 Hermes 的**持久化多 Profile 协作看板**：任务以卡片形式落在 SQLite 数据库里，在
若干状态列之间流转；一个内嵌在 Gateway 里的**调度器（dispatcher）**周期性地为就绪卡片
派生 worker agent 子进程，worker 跑完后把卡片推进到下一列。它和会话内的
[[multi-agent-architecture]] 互补——后者是"一个大脑指挥多只手"的进程内并行，Kanban 则是
**跨进程、跨 Profile、可崩溃恢复**的离线协作层。

实现全部位于 `hermes_cli/`（不在 `plugins/kanban/`，那里只有 dashboard）：

| 模块                             | 职责                                          |
| ------------------------------ | ------------------------------------------- |
| `hermes_cli/kanban_db.py`      | 数据模型、SQLite schema、`dispatch_once` 调度内核     |
| `hermes_cli/kanban.py`         | `hermes kanban ...` CLI                     |
| `hermes_cli/kanban_decompose.py` | triage 卡片自动分解（编排器）                          |
| `hermes_cli/kanban_diagnostics.py` | 受困任务的结构化诊断引擎                                |
| `hermes_cli/kanban_specify.py` | triage 卡片规格补全（不 fanout）                     |
| `hermes_cli/kanban_swarm.py`   | Swarm v1 图（并行 worker → verifier → 合成器）       |
| `tools/kanban_tools.py`        | worker/orchestrator agent 可调用的 MCP 工具       |
| `plugins/kanban/dashboard/`    | 看板可视化 dashboard 插件                          |

## 数据模型

每个看板是一个独立的 board，根目录 `kanban_home()` 下按 slug 分目录，各有自己的
`kanban.db`、workspaces、worker 日志（`board_dir` / `kanban_db_path` /
`workspaces_root` / `worker_logs_dir`）。`DEFAULT_BOARD = "default"`，board slug 受
正则 `^[a-z0-9][a-z0-9\-_]{0,63}$` 约束（`kanban_db.py:151,158`）。

### 状态列

任务在九个状态间流转（`kanban_db.py:97`）：

```python
VALID_STATUSES = {"triage", "todo", "scheduled", "ready",
                  "running", "blocked", "review", "done", "archived"}
```

```text
triage ──decompose/specify──▶ todo ──父任务全 done──▶ ready
                                                        │ dispatcher claim + spawn
                                                        ▼
   done ◀──complete── review ◀──worker 开 PR── running ──┤
                                                        ├──block──▶ blocked
                                                        └─schedule─▶ scheduled
```

`recompute_ready` 把所有父任务均为 `done` 的 `todo` 卡片提升为 `ready`
（`kanban_db.py:2005`）。`task_links` 表存父子依赖，建边时 `_would_cycle` 拒绝环。

### 任务字段

`tasks` 表（`kanban_db.py:809`）的核心字段：`id`、`title`、`body`、`assignee`（Profile 名）、
`status`、`priority`、`workspace_kind`（`scratch` / `worktree` / `dir`）、`workspace_path`、
`branch_name`、`claim_lock` / `claim_expires`（原子认领锁）、`worker_pid`、
`consecutive_failures`（统一失败计数器）、`last_failure_error`、`max_runtime_seconds`、
`last_heartbeat_at`、`current_run_id`、`skills`（强制加载的技能 JSON 数组）、
`model_override`（每任务模型覆盖）、`max_retries`（每任务熔断阈值）、`session_id`、`tenant`。

每次调度器认领任务都会在 `task_runs` 表新建一条 **run**——一次执行尝试。claim/PID/心跳/
结构化 handoff summary 都挂在 run 上，而不是 task 上；任务重试时会有多条 run
（`kanban_db.py:732`）。

## 调度器（dispatch_once）

调度器是模块级函数 `dispatch_once`（`kanban_db.py:4595`），由 Gateway 的
`_kanban_dispatcher_watcher` 每 `dispatch_interval_seconds`（默认 60 秒）调用一次。
受 `kanban.dispatch_in_gateway` 开关控制（默认 `true`，`gateway/run.py:5032`）。Gateway
每个 tick 会**遍历所有未归档 board**，逐个 `dispatch_once`，所以新建 board 无需重启。

单次 tick 的步骤（`kanban_db.py:4609`）：

1. `release_stale_claims` — 回收 TTL 过期的 running 任务（默认 claim TTL 15 分钟）。
2. `detect_stale_running` — 回收心跳停滞（`_STALE_HEARTBEAT_GAP_SECONDS = 3600`）的任务。
3. `detect_crashed_workers` — 本机 PID 已消失的 worker 视为崩溃；清洁退出但没调用
   `kanban_complete`/`kanban_block` 的视为协议违规，直接 auto-block。
4. `enforce_max_runtime` — 超出 `max_runtime_seconds` 的 worker 被 SIGTERM（再 SIGKILL）。
5. `recompute_ready` — 提升满足依赖的 `todo → ready`。
6. 对每个 `ready` 且 unclaimed 的卡片，按 `priority DESC, created_at ASC` 排序，原子
   `claim_task` 后调用 `spawn_fn(task, workspace, board)`。
7. **review 列分发** — 被 worker 推进到 `review` 的卡片（已开 PR），调度器派生一个
   review agent（强制加载 `sdlc-review` 技能），验证 PR 并 merge（→ done）或打回
   （→ running）。review spawn 与 ready spawn 共享同一并发预算。

### worker 派生

`_default_spawn`（`kanban_db.py:5160`）以 fire-and-forget 方式启动子进程
`hermes -p <assignee> --accept-hooks [--skills kanban-worker] [-m <model_override>] chat -q "work kanban task <id>"`。
关键点：

- 通过环境变量把 board 上下文钉死给子进程：`HERMES_KANBAN_DB`、`HERMES_KANBAN_BOARD`、
  `HERMES_KANBAN_TASK`、`HERMES_KANBAN_WORKSPACE`、`HERMES_KANBAN_BRANCH`、
  `HERMES_KANBAN_RUN_ID`、`HERMES_KANBAN_CLAIM_LOCK`——worker 不会误看其它 board。
- 自动追加 `--skills kanban-worker`（若该技能在 worker 的 HERMES_HOME 下可解析）；
  `task.skills` 里的每个技能再各自 `--skills X` 追加。
- `model_override` 存在时透传 `-m`，覆盖 Profile 默认模型。
- workspace 由 `resolve_workspace` 解析，`worktree` 类型卡片在独立 git worktree /
  分支上工作，详见 [[worktree-isolation]]。

### 并发控制

调度器有三层独立的并发闸门（均从 `config.yaml` 的 `kanban.*` 读取）：

| 配置                            | 语义                                                        |
| ----------------------------- | --------------------------------------------------------- |
| `kanban.max_spawn`            | **活跃并发上限**——计入已 `running` 的任务 + 本 tick 的 spawn，全局封顶 worker 数 |
| `kanban.max_in_progress`      | `running` 数达到该值则本 tick 完全跳过 spawn（让慢 worker 先收尾）           |
| `kanban.failure_limit`        | 连续失败熔断阈值（默认 `DEFAULT_FAILURE_LIMIT = 2`），可被任务级 `max_retries` 覆盖 |

`max_spawn` 是**活跃并发上限**而非每 tick 预算——因为 `running` 任务只在 worker 调用
`kanban_complete`/`kanban_block` 或 TTL 回收时才离开该状态，若按每 tick 预算解释，60 秒
tick 会让并发无界增长（`kanban_db.py:4623` 注释）。

`consecutive_failures` 是统一计数器：spawn 失败、超时、崩溃都会 +1，**只有成功完成才清零**
（`complete_task`）。超过 `failure_limit` 时 `_record_task_failure` 触发熔断，把任务
auto-block，避免调度器永远在不可修复的任务上空转。

### 重生守卫（respawn guard）

`check_respawn_guard`（`kanban_db.py:4469`）在每次 claim 前检查，命中则本 tick 推迟 spawn：

- `blocker_auth` — 上次失败错误匹配配额 / 鉴权模式，立即重试无意义。
- `recent_success` — `_RESPAWN_GUARD_SUCCESS_WINDOW = 3600` 秒内有成功 run，等人审查。
- `active_pr` — `_RESPAWN_GUARD_PR_WINDOW = 86400` 秒内的评论里出现 GitHub PR URL，
  重生会重复开 PR。

被守卫的任务留在 `ready`，并写 `respawn_guarded` 事件供 `hermes kanban tail` 诊断。

## 编排器：triage 自动分解

`kanban_decompose.py` 实现"编排器"角色。卡片以 `--triage` 创建后落在 `triage` 列，Gateway
dispatcher 的 `_auto_decompose_tick` 在分发前先处理它们，受 `kanban.auto_decompose`（默认
`true`）控制，每 tick 上限 `kanban.auto_decompose_per_tick`（默认 3，`gateway/run.py:5256`）。

分解器把用户的 Profile 名册（含描述）喂给辅助 LLM，要求返回一个任务图 JSON：每个子任务
带 `title`/`body`/`assignee`/`parents`（指向同列表的 0-based 索引，表达数据依赖）。
`decompose_triage_task`（`kanban_db.py:3126`）在**单个 write 事务**里原子地：用 Kahn 拓扑
排序检测环、创建全部子卡、按 `parents` 建依赖边、把根卡片从 `triage` 翻到 `todo`。

根任务**不消失**，它成为所有叶子子任务的父节点——当整张图完成时，根卡片重新被提升为
`ready`，其 assignee（编排器 Profile）醒来判断是否收工或追加更多任务。`fanout=false`
则退化为 `kanban specify` 的效果：只收紧 body 并 `triage → todo`，不建子任务。

## 诊断引擎

`kanban_diagnostics.py` 是无状态、只读的规则引擎，按需在 dashboard `/board` 加载或
`hermes kanban diagnostics` 时对 `(task, events, runs)` 跑 `_RULES`，产出结构化
`Diagnostic`（带 `kind` / `severity` ∈ {warning, error, critical} / 建议恢复动作）。
当前规则（`kanban_diagnostics.py:919`）：`hallucinated_cards`、`triage_aux_unavailable`、
`prose_phantom_refs`、`repeated_failures`、`repeated_crashes`、`stuck_in_blocked`、
`stranded_in_ready`。每个诊断附带至少一个可执行的恢复动作（reclaim / reassign / unblock /
cli_hint / comment），dashboard 渲染成按钮，CLI 渲染成提示。

## MCP 工具

worker / orchestrator agent 通过 `tools/kanban_tools.py` 注册的工具操作看板。注册的工具名
（`kanban_tools.py:1218`）：

| 工具                | 用途                                     |
| ----------------- | -------------------------------------- |
| `kanban_show`     | 查看任务（含评论 + 事件 + run）                    |
| `kanban_list`     | 列任务（仅 orchestrator 模式可用）               |
| `kanban_create`   | 创建子任务                                  |
| `kanban_complete` | 标记完成（带 summary / metadata / artifacts） |
| `kanban_block`    | 标记受阻并附原因                               |
| `kanban_comment`  | 追加评论                                   |
| `kanban_heartbeat`| 上报心跳，防止被判定 stale                       |
| `kanban_link`     | 添加父子依赖边                                |
| `kanban_unblock`  | 把受阻任务退回 ready（仅 orchestrator）          |

工具按 worker / orchestrator 两种模式分级——`_require_orchestrator_tool` 让 `kanban_list` /
`kanban_unblock` 等只对编排器开放，worker 只能操作自己认领的任务（`_enforce_worker_task_ownership`）。

## CLI 子命令

`hermes kanban <action>`（`kanban.py:191` 的 `build_parser`）：

```text
init                  创建 kanban.db（幂等）
boards list|create|remove|switch|current|rename|set-workdir
create                创建任务（--triage / --workspace / --branch / --assignee
                      / --skill / --model-override / --max-retries / --max-runtime）
swarm                 创建 Swarm v1 图（并行 worker → verifier → synthesizer）
list (ls) / show / assign / reclaim / reassign / diagnostics
link / unlink / claim / comment / complete / edit / block
schedule / unblock / archive / tail / runs / heartbeat / log / gc
dispatch              手动跑一次调度 tick（--dry-run / --max / --failure-limit）
daemon                已废弃——调度器现在内嵌在 gateway
watch / stats         实时事件流 / 状态统计
notify-subscribe|notify-list|notify-unsubscribe   网关订阅终态事件
specify / decompose   triage 卡片规格补全 / 自动分解
```

## 网关通知与 ACP

`kanban_notify_subs` 表把 Gateway 来源（platform + chat + thread）订阅到任务终态事件。
Gateway 的 `_kanban_notifier_watcher` 每 5 秒 tail `task_events`，把 `completed` /
`blocked` / `spawn_auto_blocked` 推回原请求者，闭合 human-in-the-loop 回路。任务在 ACP
会话内创建时会把 `HERMES_SESSION_ID` 写入 `session_id` 字段，从而支持按会话渲染看板。

## 相关页面

- [[multi-agent-architecture]] — 会话内多 Agent；Kanban 是其跨进程、可恢复的对照层
- [[cron-scheduling]] — 同样由 Gateway tick 驱动的自动化机制
- [[worktree-isolation]] — `worktree` 类型卡片的 git 隔离工作区
- [[mcp-and-plugins]] — `kanban_*` 工具的注册与 dashboard 插件
- [[configuration-and-profiles]] — 任务 assignee 即 Profile，worker 以 `hermes -p` 启动

## 相关文件

- `hermes_cli/kanban_db.py` — 数据模型、schema、`dispatch_once` 调度内核、`_default_spawn`
- `hermes_cli/kanban.py` — `hermes kanban` CLI 子命令树
- `hermes_cli/kanban_decompose.py` — triage 卡片自动分解（编排器）
- `hermes_cli/kanban_diagnostics.py` — 受困任务诊断规则引擎
- `hermes_cli/kanban_specify.py` — triage 卡片规格补全
- `hermes_cli/kanban_swarm.py` — Swarm v1 图构造
- `tools/kanban_tools.py` — worker/orchestrator 的 MCP 工具
- `gateway/run.py` — 内嵌调度器 `_kanban_dispatcher_watcher` 与通知 watcher
- `plugins/kanban/dashboard/plugin_api.py` — 看板 dashboard 插件
