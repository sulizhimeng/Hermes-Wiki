---
title: Skills System Architecture
created: 2026-04-07
updated: 2026-05-22
type: concept
tags: [skill, architecture, module, prompt-builder, curator, bundles]
sources: [tools/skills_tool.py, tools/skill_manager_tool.py, tools/skills_hub.py, tools/skills_guard.py, tools/skill_usage.py, tools/skill_provenance.py, run_agent.py, agent/prompt_builder.py, agent/curator.py, agent/curator_backup.py, agent/skill_bundles.py, hermes_cli/curator.py, hermes_cli/plugins.py, agent/skill_utils.py]
---

# 技能系统架构

## 概述

Hermes Agent 的技能系统是一个**渐进式披露（Progressive Disclosure）**架构，灵感来自 Anthropic 的 Claude Skills 系统。核心理念是：只在需要时加载完整指令，平时只保留轻量元数据，以节省 token 预算。

## 核心组件

### 1. 工具层 (`tools/skills_tool.py`)

提供两个工具：
- **`skills_list`** — 返回技能元数据列表（名称、描述、分类），token 效率高
- **`skill_view`** — 加载完整技能内容（SKILL.md + 可选引用文件）

### 2. Prompt 构建层 (`agent/prompt_builder.py`)

在每次系统提示构建时：
- 扫描 `~/.hermes/skills/` 目录
- 解析每个 SKILL.md 的 YAML frontmatter
- 构建技能索引清单注入到系统提示中
- 使用 [[prompt-builder-architecture]] 缓存结果

### 3. 技能目录结构

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md              # 主指令文件（必需）
│   ├── references/           # 支持文档
│   │   ├── api.md
│   │   └── examples.md
│   ├── templates/            # 输出模板
│   └── assets/               # 补充文件（agentskills.io 标准）
└── category/                 # 分类目录
    └── another-skill/
        └── SKILL.md
```

### 4. SKILL.md 格式

```yaml
---
name: skill-name                    # 必需，最多 64 字符
description: Brief description      # 必需，最多 1024 字符
version: 1.0.0                      # 可选
license: MIT                        # 可选
platforms: [macos]                  # 可选 — 限制 OS 平台
prerequisites:                      # 可选 — 运行时要求
  env_vars: [API_KEY]               #   环境变量
  commands: [curl, jq]              #   命令检查
setup:                              # 可选 — 交互式设置
  help: "Get key at https://..."    #   帮助文本
  collect_secrets:                  #   密钥收集
    - env_var: API_KEY
      prompt: "Enter your API key"
      secret: true
metadata:                           # 可选
  hermes:
    tags: [fine-tuning, llm]
    related_skills: [peft, lora]
---

# Skill Title

Full instructions and content here...
```

## 技能发现流程

```python
# 1. 获取所有技能目录
get_all_skills_dirs() → [Path, Path, ...]

# 2. 解析每个 SKILL.md 的 frontmatter
parse_frontmatter(raw_content) → (dict, body)

# 3. 检查平台兼容性
skill_matches_platform(frontmatter) → bool

# 4. 提取条件激活规则
extract_skill_conditions(frontmatter) → {
    "requires_tools": [...],
    "requires_toolsets": [...],
    "fallback_for_tools": [...],
    "fallback_for_toolsets": [...]
}

# 5. 构建技能索引注入到系统提示
_build_skills_index(available_tools, available_toolsets) → str
```

## 条件激活机制

技能可以根据当前可用的工具/工具集条件性显示：

- **`requires_tools`** — 需要特定工具才显示
- **`requires_toolsets`** — 需要特定工具集才显示
- **`fallback_for_tools`** — 当主工具可用时隐藏（作为备选）
- **`fallback_for_toolsets`** — 当主工具集可用时隐藏

## 平台过滤

通过 `platforms` frontmatter 字段限制技能只在特定 OS 上加载：
- `macos` → `sys.platform == "darwin"`
- `linux` → `sys.platform == "linux"`
- `windows` → `sys.platform == "win32"`

## 插件命名空间技能（2026-04-14）

除了 `~/.hermes/skills/` 的扁平目录扫描,插件还可以注册**带命名空间的技能**,避免与内置技能重名冲突。

### 注册方式

```python
# 插件的 __init__.py
def register(ctx):
    ctx.register_skill(
        name="deploy",
        path=Path(__file__).parent / "skills" / "deploy" / "SKILL.md",
        description="Deploy a service to production",
    )
```

`PluginContext.register_skill()` 内部把它存为 `{plugin_name}:{name}` 格式的 qualified name,例如插件 `myops` 注册的 `deploy` 技能实际名字是 `myops:deploy`。

**校验规则**(`hermes_cli/plugins.py:267`):
- `name` 不能含 `:`(命名空间由插件名自动派生)
- `name` 必须匹配 `[a-zA-Z0-9_-]+`
- `path` 指向的 SKILL.md 必须存在

### 调度逻辑

`skill_view(name)` 在 `tools/skills_tool.py:822` 检测 `:` 分隔符:
- **带 `:` 的名字** → `parse_qualified_name(name)` → 路由到 `_serve_plugin_skill(namespace, bare)`
- **裸名字** → 继续走原有的 `~/.hermes/skills/` 扁平树扫描

插件技能加载时会跑完整防护:
1. 插件被 disable → 返回错误(含 `hermes plugins enable` 提示)
2. 平台不匹配(`skill_matches_platform`) → 返回 UNSUPPORTED
3. 注入模式扫描(`_INJECTION_PATTERNS`) → 记日志但仍加载(与本地技能一致)
4. 返回时附带 **bundle context banner**,列出同一插件的其他技能供 agent 参考

### 不进系统提示索引

**关键区别**:插件技能**不出现在** system prompt 的 `<available_skills>` 清单里,它们是**显式 opt-in**——agent 必须知道名字(通过文档或插件 README)才能调用 `skill_view("myops:deploy")`。

这样设计的原因:
- 避免插件污染主提示词(系统提示已经很大)
- 避免 prefix cache 因为第三方插件数量波动而失效
- 用户装了什么插件,agent 不应该自动全部感知

### 相关 API

| 符号 | 位置 | 用途 |
|---|---|---|
| `PluginContext.register_skill()` | `hermes_cli/plugins.py:267` | 插件注册入口 |
| `PluginManager._plugin_skills` | `hermes_cli/plugins.py` | 注册表存储 |
| `parse_qualified_name()` | `agent/skill_utils.py:451` | 分解 `ns:bare` |
| `is_valid_namespace()` | `agent/skill_utils.py` | 命名空间合法性校验 |
| `_serve_plugin_skill()` | `tools/skills_tool.py:718` | 加载 + 防护 + banner |
| `_INJECTION_PATTERNS` | `tools/skills_tool.py`(模块级) | 与本地技能共享的注入检测

## 密钥管理

技能可以声明需要的环境变量，系统会：
1. 检查 `~/.hermes/.env` 是否已设置
2. 如果缺失且在 CLI 模式，通过回调交互式收集
3. 在 Gateway 模式，提示用户手动配置
4. 保存后持久化到 `.env` 文件

## 自动 Skill Review（Background Review）

Hermes 不只被动使用 Skill，还能**自主创建和更新 Skill**。这是 Hermes 的"自我进化"机制。

### 触发条件

三个条件同时满足时触发：

```python
if (self._skill_nudge_interval > 0                          # 功能未禁用
        and self._iters_since_skill >= self._skill_nudge_interval  # 工具调用累计达标
        and "skill_manage" in self.valid_tool_names):        # skill_manage 工具可用
```

```yaml
# config.yaml
skills:
  creation_nudge_interval: 15   # 每累计 15 次工具调用触发一次 review（0 = 禁用）
```

注意：计数器累加的是**工具循环次数**（不是对话轮次），跨轮次持续累加。agent 主动调用 `skill_manage` 时计数器归零。

### 执行流程

```text
工具调用累计达到 15 次
    ↓
轮次结束后，派生后台 agent（独立线程，max_iterations=8）
    ↓
后台 agent 拿到完整对话快照，审查：
  "有没有经过试错、调整方向、或用户期望不同做法的非平凡经验？"
    ↓
三种结果：
  ├── 有现成 skill → 调用 skill_manage 更新
  ├── 没有但值得新建 → 调用 skill_manage 创建
  └── 没什么值得存的 → "Nothing to save." 结束
    ↓
终端打印：💾 Skill "docker-network-debug" created
```

### 设计特点

- **不阻塞用户**：在回复用户之后才启动，不占用对话延迟
- **不修改主对话**：后台 agent 独立运行，不影响主 agent 的消息历史
- **共享记忆存储**：后台 agent 与主 agent 共享 `_memory_store`，skill 写入立即可用
- **与 Memory Nudge 可合并**：当 skill review 和 memory review 同时触发时，使用合并 prompt 一次处理

### 与手动创建的区别

| | 手动创建（用户指令） | 自动创建（Background Review） |
|---|---|---|
| 触发方式 | 用户说"帮我创建一个 skill" | 系统计数器自动触发 |
| 内容来源 | 用户指定 | 后台 agent 从对话中提炼 |
| 质量 | 用户控制 | agent 自主判断，可能创建也可能跳过 |
| LLM 消耗 | 主对话的一部分 | 额外消耗（后台 agent 最多 8 轮迭代） |

## Curator — 自治技能维护（v2026.4.23 引入，v0.12+ 全面重构到 1781 行）

`agent/curator.py`（1781 行）+ `agent/curator_backup.py` + `hermes_cli/curator.py`（582 行）。从"被动清理过期 skill"演进到 **active umbrella consolidation + 多层安全网（backup / pin / dry-run）+ 统一 aux 模型路由**。

### 不变量（load-bearing invariants）

- **永不触碰** bundled 或 hub-installed 技能（`.bundled_manifest` + `.hub/lock.json` 双过滤，`tools/skill_usage.py:159-217`）
- **永不自动删除** —— 只归档（`.archive/` 子目录），可 `hermes curator restore <skill>` 恢复
- **Pinned skills 跳过所有自动转换**：`tools/skill_manager_tool.py:_pinned_guard()` 在 `skill_manage` 写入路径拦截
- 使用 aux client，**永不污染主 session 的 prompt cache**
- 跑前**自动 tarball 快照** `~/.hermes/skills/.curator_backups/{TS}/`，可 `hermes curator rollback`

### 统一 auxiliary.curator 配置（v0.12, `agent/curator.py:1557-1620`）

```yaml
auxiliary:
  curator:
    provider: openrouter
    model: anthropic/claude-haiku-4-5
    api_key: ...
    base_url: ...
    timeout: 300
```

- 主路径：`auxiliary.curator.*`
- 兼容回退：`curator.auxiliary.{provider,model}`（带 deprecation warning）
- 都没配 → fall through 到主 chat model
- 与 `hermes model` CLI + dashboard Models tab 完全整合

### Consolidated vs Pruned 分类（v0.12, `agent/curator.py:498-611, 1021-1049`）

报告显式区分两类归档：

| 类型 | 含义 |
|------|------|
| **Consolidated** | 内容被合并进 umbrella skill，归档但内容保留 |
| **Pruned** | 纯过期，无目标 umbrella，直接归档 |

三步启发式分类：

1. 解析模型的结构化 YAML 块（`consolidations:` / `prunings:` + rationale）
2. 工具调用启发式（detect `write_file`/`patch`/`create` 引用已删 skill）
3. Reconcile：模型给意图，启发式验证（umbrella 存在 + 内容保留）

`_extract_absorbed_into_declarations()` 读取 `skill_manage` delete 调用上的 `absorbed_into` 参数 ground truth。

### 触发与执行

默认开启，**inactivity-triggered**（无 cron 守护进程）：CLI / Gateway 启动时检查：

1. 上次跑 > `interval_hours`（默认 `DEFAULT_INTERVAL_HOURS = 24 * 7` 即 7 天，`agent/curator.py:56`）
2. agent 闲置 > `min_idle_hours`（默认 `DEFAULT_MIN_IDLE_HOURS = 2`，`agent/curator.py:57`）
3. 状态机阈值：`DEFAULT_STALE_AFTER_DAYS = 30`（line 58）/ `DEFAULT_ARCHIVE_AFTER_DAYS = 90`（line 59）

Gateway 也 hook 进 cron-ticker 定期检查。

### 状态机（pure auto, 无 LLM）

```
active ──≥ stale_after_days (默认 30)─→ stale ──≥ archive_after_days (默认 90)─→ archived
   ↑                                                                              │
   └────────────────── 重新使用 ───────────────────────────────────────────────────┘
```

转换在 `agent/curator.py:256-296`。Pinned skill 永远跳过自动转换。

### Umbrella-Building Prompt（v0.12, `agent/curator.py:330-445`）

LLM review pass 的指令：

- **目标**：建 class-level umbrella，禁止"一 bug 一 skill"碎片化
- **策略**：扫 PREFIX CLUSTER（同首词/同域 skill）→ 合并到 umbrella / 创建新 umbrella + archive sibling / 降级为 support files（references/templates/scripts）
- **结构化输出（必需）**：

```yaml
consolidations:
  - from: <old-skill>
    into: <umbrella>
    reason: <one sentence>
prunings:
  - name: <skill>
    reason: <one sentence>
```

- **硬规则**：bundled/hub 不可删，pinned 不可动，archives only，**基于 CONTENT 合并**（不看 use_count）

### Per-run 报告（v0.12, `agent/curator.py:452-471, 970-1070`）

每次跑写 `~/.hermes/logs/curator/{YYYYMMDD-HHMMSS}/`：

- `run.json` —— 机器可读（before/after 计数、状态转换、工具调用、分类）
- `REPORT.md` —— 人类可读叙述（auto-transitions、LLM review 总结、consolidated → umbrella map、pruned + stale 时间戳）

`state.last_report_path` 指向最新报告，`hermes curator status` 直接显示路径。

### Sidecar telemetry（`tools/skill_usage.py:1-150`）

每个 skill 维护 `.usage.json` sidecar：

- **计数器**：`use_count` / `view_count` / `patch_count`
- **时间戳**：`last_used_at` / `last_viewed_at` / `last_patched_at` / `created_at` / `archived_at`
- **生命周期**：active / stale / archived（与 pinned 正交）
- **派生**：`latest_activity_at()` 取最新（**故意排除创建**），`activity_count` 累加
- **原子写**：tempfile + `os.replace`；所有 bump 都是 best-effort，绝不破坏底层 tool

`bump_use()` 在 `skill_view` / 预加载 / skill 调用三处都触发（`#17932`），驱动状态机。

### Provenance（`tools/skill_provenance.py:1-79`）

- `get_current_write_origin()` 返回 `"background_review"`（curator fork）或 `"foreground"`（用户发起 agent）
- `skill_manage(action="create")` 自动 `mark_agent_created()`，置 `created_by: "agent"`
- Curator **只动 agent-created skill**，从不动用户手写或 hub-installed

### Dry-run（v0.12, `agent/curator.py:303-327, 1386-1405`）

```bash
hermes curator run --dry-run
```

跳过自动转换，给 LLM 加 `CURATOR_DRY_RUN_BANNER`，LLM 被指示"描述将要做什么，不做"。报告照写，`state.last_report_path` 更新，`run_count` 不增。用户读完决定提交。

### Backup / Rollback（v0.12, `agent/curator_backup.py`）

- 非 dry-run 每跑前自动打 tarball
- 包含：所有 SKILL.md + 目录 / `.usage.json` / `.archive/` / `.bundled_manifest` / `.curator_state` / cron-jobs.json 快照（仅 skill 引用部分）
- 排除：`.curator_backups/` 自身、`.hub/`（保 hub 纯净）
- 位置：`~/.hermes/skills/.curator_backups/{YYYYMMDD-HHMMSS-NN}/{tar.gz, manifest.json}`
- `hermes curator rollback [--id <stamp>] [-y]` 还原 skills tree + cron 中的 skill 引用（cron 其他字段不动）
- Rollback **本身可撤销**：跑前先打 pre-rollback 快照

### CLI 全套（`hermes_cli/curator.py:39-582`）

```bash
hermes curator status                              # 健康 + 配置 + 状态分布 + pinned + 最活跃/最少用
hermes curator run [--sync|--background] [--dry-run]
hermes curator pause / resume
hermes curator pin <skill> / unpin <skill>
hermes curator archive <skill> [--reason ...]
hermes curator restore <skill>                     # 从 .archive/ 恢复
hermes curator list-archived
hermes curator prune [--days N] [--dry-run] [-y]  # 批量按日龄归档，默认 90d
hermes curator backup [--reason ...]               # 手动快照
hermes curator rollback [--list|--id <stamp>] [-y]
```

`/curator` 斜杠命令暴露同名子命令。

## Skill Bundles（v0.14, `agent/skill_bundles.py`）

YAML 文件在 `~/.hermes/skill-bundles/` 定义 bundle：一个 `/<alias>` 一击加载多 skill。

- Slash 分发先查 bundle（bundle 在名字冲突时胜出，`agent/skill_bundles.py:29-31`）
- `build_bundle_invocation_message()` 把所有引用 skill 装一条消息 + bundle header（line 253-340）
- 缺失 skill 优雅跳过 + note（line 292-295）
- 每个被调 skill `bump_use()`（line 299-302）

**`/reload-skills`**：重新扫描 `~/.hermes/skills/` 发现新装/卸载的 skill，无需重启进程。**用户发起的 rescan**——不重置 prompt cache（skills 是按需通过 `/skill-name`、`skills_list`、`skill_view` 调用，不需要常驻系统提示）。重扫后通过 next-turn note 通知 agent，每个新增/移除的 skill 附带 60 字符描述。

> 说明：原 PR 包含一个 `skills_reload` agent 工具，但在后续 refactor（`dd2d1ba5e`）中被显式删除——agent 已经能通过 `skill_view` / `skills_list` 看到磁盘上新装的 skill，不需要额外 schema surface。

**`/reload-mcp` 加确认提示**：MCP 重载会失效 prompt cache，gateway 现在弹出确认对话框（包含"未来不再询问"的 opt-out 选项），避免误操作清掉昂贵的缓存。

## 拒绝写 pinned skills（v2026.4.23+）

`tools/skill_manager_tool.py:134` 新增 `_pinned_guard(name)`，在 `skill_manage` 的 create/update/archive/delete 路径上拦截 pinned skill 修改：

```python
if rec.get("pinned"):
    return f"Skill '{name}' is pinned and cannot be modified by skill_manage..."
```

这是 Curator 不变量的延伸——pinned 状态对 agent 也是禁区，只能通过 `hermes curator unpin` 显式解锁。

## Skills Hub & 默认 taps（v0.14）

`tools/skills_hub.py:330-337` 默认 taps：

```python
DEFAULT_TAPS = [
    {"repo": "openai/skills", "path": "skills/"},
    {"repo": "anthropics/skills", "path": "skills/"},
    {"repo": "huggingface/skills", "path": "skills/"},      # v0.14 新增 trusted
    {"repo": "VoltAgent/awesome-agent-skills", "path": "skills/"},
]
```

`huggingface/skills` 享 **trusted** trust level（caution 验证通过即装）。

**Direct-URL install**（`tools/skills_hub.py:397+`）：
```bash
skills_hub install <github-raw-url>
```
任意 GitHub 仓库下载 + skills_guard 扫描。

**External dirs**（`agent/skill_utils.py:242-282`）：
```yaml
skills:
  external_dirs:
    - /path/to/team-skills
    - ~/.agents/skills
```
和 `~/.hermes/skills/` 并行扫描。

## Skills Guard — 安全扫描（`tools/skills_guard.py:1-933`）

每次 hub install 跑：

- **Trust level**：builtin（永不扫） / trusted（openai/anthropics/huggingface/skills） / community（其他）
- **Install policy**（`tools/skills_guard.py:41-51`）：safe 始终允许；caution 看 trust（trusted 放行，community 拦截）；dangerous 一律拦截（agent-created 询问确认）
- **~56 个威胁模式**（`tools/skills_guard.py:86-488`）：exfil（env vars / SSH/AWS dirs / base64+env）、injection（jailbreak / role-hijack / system prompt leak）、destructive（rm -rf / mkfs / dd）、persistence（cron / `~/.bashrc` / sudoers）、network（reverse shell / tunnel）、obfuscation（eval / base64 pipe）、supply chain（unpinned dep / 远程 fetch）、privilege escalation（sudo / setuid）、agent config tampering
- **结构检查**（`tools/skills_guard.py:738-852`）：文件数、总大小、二进制、symlink escape、可执行位
- **不可见 unicode 检测**（`tools/skills_guard.py:581-594`）：零宽连接符、方向覆盖、用于注入的 BOM
- **内容哈希**：skill 目录 SHA-256，完整性追踪

## 相关页面

- [[prompt-builder-architecture]] — 技能索引构建与条件激活
- [[skills-and-memory-interaction]] — 技能与记忆的交互设计
- [[security-defense-system]] — 技能安全扫描与信任级别策略
- [[kanban-multi-agent-board]] — 多 Agent 协作时配合 skills 分发

## 相关文件

- `tools/skills_tool.py` — 技能工具实现（1378 行）
- `agent/prompt_builder.py` — Prompt 构建与技能索引
- `agent/skill_utils.py` — 技能解析工具函数
- `agent/skill_commands.py` — 技能斜杠命令
- `tools/skills_sync.py` — 技能同步机制
- `tools/skills_hub.py` — 技能中心（搜索/安装）
- `tools/skill_manager_tool.py` — 技能管理工具
