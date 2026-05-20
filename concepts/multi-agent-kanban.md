---
title: 多 Agent Kanban 板架构
created: 2026-05-20
updated: 2026-05-20
type: concept
tags: [multi-agent, kanban, delegation, orchestration]
source_files:
  - hermes_cli/kanban_db.py
  - hermes_cli/kanban.py
  - hermes_cli/kanban_swarm.py
  - hermes_cli/kanban_decompose.py
  - hermes_cli/kanban_diagnostics.py
  - hermes_cli/kanban_specify.py
  - tools/kanban_tools.py
verified_against: hermes-agent HEAD (2026-05-20)
---

# 多 Agent Kanban 板（Tenacity Release / v0.13.0）

Hermes 历史上有四种 Agent 协作机制（[[multi-agent-architecture]] 第一版）：

1. `delegate_task`（单进程同步 fork，深度 ≤ 1）
2. `mixture_of_agents`（多模型聚合）
3. Background Review fork（同步 review）
4. `send_message`（跨平台消息投递，非 agent 间通信）

**Kanban 是第五种**，也是第一个真正"分布式异步"的 —— 多个 worker 进程跨机器 / 跨 backend 协作，靠 SQLite 板做协调，自带心跳 / 回收 / 重试 / 诊断。

---

## 1. 设计目标

| 目标 | 实现 |
|------|------|
| **能跨 profile 协作** | 板存在共享根 `<hermes_root>/kanban.db`，profile 不是隔离边界 |
| **能跨 backend 协作** | worker 用**工具调用**（不是 shell）操作板，Docker/Modal/SSH worker 也能更新 |
| **能容忍崩溃** | 心跳 + 死锁回收 + spawn 崩溃循环检测 |
| **有限重试** | 每任务可配 `max_retries`，超限自动归档 |
| **可诊断** | 规则引擎从 (task, events, runs) 推导可恢复信号 |
| **不重新发明 scheduler** | 把图原语映射到既有 task / link / event / run 表，dashboard / 通知器 / dispatcher 不动 |

---

## 2. 存储模型

源码：`hermes_cli/kanban_db.py:809` 起 6 张表。

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id TEXT PRIMARY KEY,
    title TEXT, body TEXT,
    assignee TEXT,            -- profile 名 / 工作流模板键
    status TEXT,              -- todo|ready|running|scheduled|blocked|done|archived
    priority INTEGER,
    tenant TEXT,              -- 隔离命名空间
    workspace_kind TEXT,      -- "cwd" | "git" | "worktree" | ...
    workspace_path TEXT,
    branch_name TEXT,
    created_by TEXT,
    created_at INTEGER,
    started_at INTEGER,
    completed_at INTEGER,
    result TEXT,
    skills TEXT,              -- JSON list
    max_retries INTEGER,
    session_id TEXT,          -- 让 /resume 找回
    workflow_template_id TEXT,
    current_step_key TEXT,
    ...
);
CREATE TABLE IF NOT EXISTS task_links     (...);  -- parent/child + depends-on
CREATE TABLE IF NOT EXISTS task_comments  (...);  -- 黑板 + decompose 元数据 JSON
CREATE TABLE IF NOT EXISTS task_events    (...);  -- 审计 + diagnostics 输入
CREATE TABLE IF NOT EXISTS task_runs      (...);  -- 每次 spawn 的运行记录（PID/退出码/runtime）
CREATE TABLE IF NOT EXISTS kanban_notify_subs(...);
```

**多板**（用户场景：一个仓库一个板）：

```
<hermes_root>/
├── kanban.db                            # 默认板
└── kanban/
    └── boards/<slug>/
        ├── kanban.db                    # 命名板
        ├── workspaces/                  # worker 各自 worktree
        └── logs/<run_id>/               # 每次 spawn 的 stdout/stderr
```

worker 启动时传 `--board <slug>`，**只能枚举自己板的任务**，dispatcher 的 tick 也只触及对应板的 DB。

---

## 3. 任务状态机

```
       (decompose)             (assignee 抢)
triage ──────────► todo ────► ready ────► running
                                            │
                                            ├─ heartbeat 超时 → 回 ready（重试 ++）
                                            ├─ block            → blocked → ready
                                            ├─ complete         → done
                                            └─ 超 max_retries   → archived
```

`triage` 是分解前的 root state（见第 5 节自动分解）。

---

## 4. Worker 端工具表面

源码：`tools/kanban_tools.py`，9 个工具注册到 schema，**默认不可见**：

```python
# 门控（kanban_tools.py:62）
def _check_kanban_mode() -> bool:
    return os.environ.get("HERMES_KANBAN_TASK") is not None

# 或者 profile 配置 orchestrator
def _check_kanban_orchestrator_mode() -> bool: ...
```

只有：

- dispatcher 派生的 worker 进程（`HERMES_KANBAN_TASK=<id>`）
- 显式启用 `kanban` toolset 的 orchestrator profile

才看到这些工具。`hermes chat` 普通 session 看到的是 **0 个** kanban tool（合理：用户不应能误调用）。

工具列表（`tools/kanban_tools.py:1219` 起注册）：

| 工具 | 用途 |
|------|------|
| `kanban_show` | 看自己 task 状态/子任务/comments |
| `kanban_list` | 按 status / assignee 过滤 |
| `kanban_complete` | 提交结果，触发依赖任务 |
| `kanban_block` | 标 blocked + 原因 |
| `kanban_unblock` | 解 blocked |
| `kanban_heartbeat` | **必须周期调用**，否则被回收 |
| `kanban_comment` | 写黑板（结构化 JSON） |
| `kanban_create` | 派生子任务 |
| `kanban_link` | 加 depends-on 边 |

> 为什么用工具不是 shell：(1) 跨 backend 可达；(2) 跳过 shlex 引号坑；(3) 错误以 JSON 返回而不是 stderr 字符串。

`_enforce_worker_task_ownership(tid)`（line 132）保证 worker 只能改自己持有的任务 —— 越权返回 tool_error。

---

## 5. 自动分解（`hermes kanban decompose`）

源码：`hermes_cli/kanban_decompose.py`，477 行。

- 拿 profile roster（含 description）和默认 fallback profile 喂辅助 LLM。
- 让 LLM 返回任务图 JSON（节点 + parent/child + depends-on）。
- 原子创建子任务 + 链接 + 把 root 翻 `triage → todo`。
- root 任务**保留**：所有 leaf 完成后，root 唤醒，其 assignee（orchestrator profile）评估是否真完成；不够就再加任务。

退化形式：`hermes kanban specify`（`kanban_specify.py`）—— `fanout=false`，只把 root body 修一修翻 `triage → todo`，不生子任务。

---

## 6. Swarm 拓扑

源码：`hermes_cli/kanban_swarm.py`。

不引入第二个 scheduler，直接写入任务图：

```
planning root (completed immediately)
    ├─ parallel specialist workers (status=ready)
    └─ verifier (status=todo until all workers done)
         └─ synthesizer (status=todo until verifier done)
```

黑板是 root 任务的 JSON `task_comments`。dispatcher / dashboard / 通知器**不知道也不需要知道** swarm 拓扑 —— 它们只看 task / link 表。

---

## 7. 诊断

源码：`hermes_cli/kanban_diagnostics.py`，1058 行。

`Diagnostic` 数据类字段：

```python
@dataclass
class Diagnostic:
    kind: str           # canonical code，UI / 测试匹配用
    severity: str       # warning | error | critical
    title: str          # 一行人类描述
    detail: str
    suggested_actions: list[DiagnosticAction]  # dashboard 按钮 / CLI 提示
```

**规则只读**，从 (task, recent events, recent runs) 推导，不写 DB。dashboard `/board` / `/tasks/:id` / CLI `hermes kanban diagnostics` 共用同一组规则。

规则覆盖：

- 幻觉 task ID（agent 引用了不存在的 task）
- spawn crash loop（同一 task 连续 spawn 全部立即崩）
- 长时间 blocked / running 但无 heartbeat
- 缺失配置（assignee profile 不存在）

明确**不覆盖**：单次 provider 502 等 transient — 那不是 operator 可解的诊断。

自动清除：底层失败模式消失后规则停发；audit trail 在 `task_events` 永留。

---

## 8. CLI

`hermes kanban` 完整子命令族（来自 `kanban.py` `build_parser`）：

```
hermes kanban add <title> [--assignee X --priority P --workspace ...]
hermes kanban show <task_id>
hermes kanban list [--status running --assignee X --tenant T --board slug]
hermes kanban take <task_id>          # 手动抢
hermes kanban complete <task_id> --result ...
hermes kanban block <task_id> --reason ...
hermes kanban unblock <task_id>
hermes kanban comment <task_id> "<txt>"
hermes kanban link <a> <b> --type depends-on
hermes kanban decompose [<task_id> | --all]
hermes kanban specify <task_id>
hermes kanban diagnostics [<task_id>]
hermes kanban swarm "<plan>"
hermes kanban serve                   # dispatcher 守护
hermes kanban boards [list|create|switch]
```

`/kanban ...`（斜杠命令）解析同一 argparse 表面。

---

## 9. 与现有机制的关系

| 场景 | 用哪个 |
|------|--------|
| **一次性子任务，不需跨 session 持久化** | `delegate_task`（[[multi-agent-architecture]]） |
| **多模型聚合一道题** | `mixture_of_agents` |
| **多 agent 长期协作，需要崩溃恢复** | **Kanban** |
| **跨平台投递消息** | `send_message`（gateway） |
| **同一 session 内自动维持目标** | `/goal`（[[goal-loop-and-steering]]） |

Kanban 是给"团队成员长期跨 session 协作"准备的 —— 不是给"agent 在一个 turn 里 fork 3 个并行工作"准备的。后者继续走 `delegate_task`。

---

## 10. 不变量

- profile 不是隔离边界，**板才是**：跨 profile 的 worker 共享同一 `kanban.db`。
- worker 只能修自己持有的任务（`_enforce_worker_task_ownership`）。
- 普通 chat session 看不到 kanban 工具（双门控：env var + orchestrator profile）。
- 规则引擎只读，不会"为了清诊断"去改 DB。
- root 任务在 leaf 完成后唤醒，给 orchestrator 评估是否真完成 —— 不是简单的 fan-in。
- 诊断 auto-clear：失败模式消失 → 规则停发，audit 留存。
