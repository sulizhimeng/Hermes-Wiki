---
title: Agent Loop and Prompt Assembly
created: 2026-04-07
updated: 2026-05-19
type: concept
tags: [agent-loop, prompt-builder, architecture, component]
sources: [hermes-agent 源码分析 2026-05-19 (v0.14.0)]
---

# Agent Loop and Prompt Assembly

## AIAgent 核心循环

v0.14.0 起 `run_agent.py` 经历大规模重构：`AIAgent` 类**仍定义在 `run_agent.py`**（类体起 `run_agent.py:326`），但绝大多数实现已抽取到 `agent/` 子模块，类内只保留**薄转发器（forwarder）方法**。

```python
# run_agent.py:326
class AIAgent:
    def __init__(self,
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        api_mode: str = None,           # 例如 "codex_app_server"
        model: str = "",               # 默认空字符串，运行时解析为 anthropic/claude-opus-4.6
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        save_trajectories: bool = False,
        platform: str = None,           # "cli", "telegram", etc.
        session_id: str = None,
        skip_context_files: bool = False,
        load_soul_identity: bool = False,
        skip_memory: bool = False,
        iteration_budget: "IterationBudget" = None,
        credential_pool=None,
        checkpoints_enabled: bool = False,
        # ... 共 60+ 参数
    ):
        """Forwarder — see agent.agent_init.init_agent"""
        from agent.agent_init import init_agent
        init_agent(self, ...)

    def chat(self, message: str, stream_callback=None) -> str:
        """简单接口 — 返回最终响应字符串（内部调用 run_conversation）"""

    def run_conversation(self, user_message, system_message=None,
                         conversation_history=None, task_id=None,
                         stream_callback=None, persist_user_message=None) -> dict:
        """Forwarder — see agent.conversation_loop.run_conversation
        完整接口 — 返回 dict {final_response, messages}"""
```

`__init__` 本身只是转发器（`run_agent.py:349`），真正的初始化逻辑在 `agent/agent_init.py` 的 `init_agent()`（1504 行）。

## 对话循环

`run_conversation` 转发到 `agent/conversation_loop.py:187` 的 `run_conversation()`（4099 行，整个项目最大的抽取模块）。循环骨架：

```python
# agent/conversation_loop.py
agent.iteration_budget = IterationBudget(agent.max_iterations)
api_call_count = 0

# codex_app_server 运行时分支（见下文）
if agent.api_mode == "codex_app_server":
    return agent._run_codex_app_server_turn(...)

while (api_call_count < agent.max_iterations
       and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
    api_call_count += 1
    # ... 发起 API 调用
    if assistant_message.tool_calls:
        agent._execute_tool_calls(assistant_message, messages,
                                  effective_task_id, api_call_count)
    else:
        return final_response
```

- 完全同步执行（不使用 asyncio）
- 消息格式遵循 OpenAI 标准：`{"role": "system/user/assistant/tool", ...}`
- 工具执行分发到 `agent/tool_executor.py`（910 行），含 `execute_tool_calls_sequential` 与 `execute_tool_calls_concurrent` 两条路径

## Codex App-Server 运行时

v0.14.0 新增 `codex_app_server` 运行时（`agent/codex_runtime.py`，448 行）。当 `api_mode == "codex_app_server"` 时，整轮对话被交给一个 `codex app-server` 子进程，再把它的事件投影回 Hermes 的 `messages` 列表，使记忆/技能复盘仍然工作。`AIAgent._run_codex_app_server_turn()`（`run_agent.py:3894`）是转发器，指向 `agent/codex_runtime.py` 的 `run_codex_app_server_turn()`。该模块还包含 `run_codex_stream`（Codex Responses API 流式）与 `run_codex_create_stream_fallback`（流式初始化失败的恢复路径）。

## 每轮文件变更校验页脚（2026-05-15）

某些模型会在一轮内发起多个并行 `patch` / `write_file` 调用，部分失败
（如 `Could not find old_string`），却在最终文本响应里声称"已修改全部
文件"。为结构性地杜绝这种过度声称，Agent 在每轮结束时追加一个**文件变更
校验页脚**（commit `c594a23`，#24498）。

### 工作方式

- **纯事后检查**：只读取本轮的工具结果，不发起新的 LLM 调用、不向历史
  注入合成消息（保持 prefix cache），也不改动工具参数派发。
- `_record_file_mutation_result()` 在每个工具调用后被调用（`blocked`
  的调用不计入）。文件变更工具集由共享模块的 `FILE_MUTATING_TOOL_NAMES`
  定义 —— 即 `{"write_file", "patch"}`（commit `c3094b4` 把它抽到
  `agent/tool_result_classification.py`，与 `file_mutation_result_landed()`
  助手一起）。
- 按路径维护本轮状态字典 `_turn_failed_file_mutations`：首个失败胜出
  （first-error-wins），对同一路径的**后续成功写入会清除该失败条目**，
  因此单文件重试恢复不会被误报。
- `_extract_file_mutation_targets()` 从工具参数解析受影响路径，覆盖
  `write_file`、`patch-replace`、`patch-v4a`（单文件与多文件）。
- 同时接入 `_execute_tool_calls_concurrent` 与
  `_execute_tool_calls_sequential`，并行批量补丁与逐个编辑都被覆盖。

### 页脚渲染与时机

`_format_file_mutation_failure_footer()` 渲染最多 10 条路径
（每条带首个错误预览），多余的折叠为 `… and N more`：

```text
⚠️ File-mutation verifier: N file(s) were NOT modified this turn despite
any wording above that may suggest otherwise. Run `git status` or
`read_file` to confirm.
  • path/to/file — [patch] Could not find old_string
```

页脚在 Agent 循环退出后、`transform_llm_output` / `post_llm_call` 插件
钩子运行**之前**追加，因此插件仍能看到并修改这段增强后的文本。仅当本轮
存在真实文本响应且用户未中断时才追加。

**开关**：`display.file_mutation_verifier`（bool，默认 true），
`HERMES_FILE_MUTATION_VERIFIER` 环境变量可覆盖配置。

## 系统提示构建

`AIAgent._build_system_prompt()` 现在是转发器（`run_agent.py:2164`），指向 `agent/system_prompt.py` 的 `build_system_prompt()`。该模块（346 行）从 v0.14.0 起从 `run_agent.py` 抽出，把系统提示组织为**三个层级（tier）**，由 `build_system_prompt_parts()` 返回 `{stable, context, volatile}` 字典，再由 `build_system_prompt()` 用 `"\n\n".join(...)` 拼成完整字符串。

按 tier 与拼装顺序（实测见 `agent/system_prompt.py:60`）：

**stable 层**（标识/指导，会话内字节稳定）：

1. **SOUL.md** — Agent 身份（`~/.hermes/SOUL.md`，不存在则用 `DEFAULT_AGENT_IDENTITY`）
2. **hermes-agent 帮助指引** — `HERMES_AGENT_HELP_GUIDANCE`
3. **工具感知行为指导** — `MEMORY_GUIDANCE` / `SESSION_SEARCH_GUIDANCE` / `SKILLS_GUIDANCE` / Kanban 指导，按已加载工具注入
4. **Computer-use 指导** — 仅当 `computer_use` 工具存在
5. **Nous 订阅块** — `build_nous_subscription_prompt()`
6. **工具使用强制指导** — 受 `agent.tool_use_enforcement` 配置控制（`auto`/`true`/`false`/列表）
7. **模型特定执行指导** — Google（gemini/gemma）注入 `GOOGLE_MODEL_OPERATIONAL_GUIDANCE`；OpenAI/Codex/Grok 注入 `OPENAI_MODEL_EXECUTION_GUIDANCE`
8. **Skills 索引** — `build_skills_system_prompt()`，仅当存在 `skills_list`/`skill_view`/`skill_manage` 工具
9. **Alibaba 模型名修正块** — 仅 `provider == "alibaba"`
10. **环境提示** — `build_environment_hints()`（WSL/Termux/容器等）
11. **平台提示** — `PLATFORM_HINTS[platform]`，未命中再查 `platform_registry`

**context 层**（cwd 相关，会话间可变）：

12. **用户/Gateway 系统消息** — 若 `run_conversation` 传入 `system_message`
13. **项目上下文文件** — `.hermes.md/HERMES.md → AGENTS.md → CLAUDE.md → .cursorrules`（first match wins）

**volatile 层**（每会话/每轮变化，从不缓存）：

14. **MEMORY 快照** — `~/.hermes/memories/MEMORY.md`（冻结）
15. **USER PROFILE 快照** — `~/.hermes/memories/USER.md`（冻结）
16. **外部 Memory Provider 块** — mem0/honcho/holographic 等，若启用
17. **时间戳行** — `Conversation started:` + 可选 Session ID / Model / Provider

**缓存机制**：系统提示在会话内只构建一次（`self._cached_system_prompt`），只在上下文压缩后才重建（`invalidate_system_prompt()`），确保每轮对话复用同一份 → LLM prefix cache 命中率最大化。三层 tier 顺序刻意把稳定内容放最前、易变内容放最后，最大化前缀命中。时间戳行只精确到**日期**（不含分钟），保证 system prompt 全天字节稳定。

**记忆冻结模式**：MEMORY.md / USER.md 的内容是**加载时的快照**，即使对话中模型写入新记忆也不会反映到当前会话的 system prompt 里——下次会话才生效。这是为保护 prefix cache 的刻意设计。

## 平台提示 (PLATFORM_HINTS)

为不同消息平台注入特定指导：

| 平台 | 关键指导 |
|------|----------|
| `telegram` | 不使用 markdown，MEDIA: 路径发送文件 |
| `discord` | 支持 markdown，文件作为附件 |
| `whatsapp` | 不使用 markdown，原生媒体 |
| `slack` | 文件作为附件 |
| `signal` | 不使用 markdown，纯文本 |
| `email` | 清晰结构化，适合邮件 |
| `cron` | 无用户在场，完全自主执行 |
| `cli` | 终端可渲染的纯文本 |
| `sms` | 纯文本，~1600 字符限制 |

## 执行指导

### 通用工具使用强制指导

```text
# Tool-use enforcement
You MUST use your tools to take action — do not describe what you would do
or plan to do without actually doing it.
```

是否注入由 `config.yaml` 的 `agent.tool_use_enforcement` 控制（`"auto"` 默认 / `true` 全部 / `false` 关闭 / 自定义模型名子串列表）。`auto` 时匹配 `TOOL_USE_ENFORCEMENT_MODELS`（`agent/prompt_builder.py:276`）：

```python
TOOL_USE_ENFORCEMENT_MODELS = ("gpt", "codex", "gemini", "gemma", "grok", "glm", "qwen", "deepseek")
```

### OpenAI 模型额外指导

`OPENAI_MODEL_EXECUTION_GUIDANCE` 适用于 `gpt` / `codex` 模型，**同样也应用于 xAI `grok` 模型**（`system_prompt.py:162`）——grok 表现出相同的失败模式（未调用工具就声称完成、用替代方案而非现有工具、回复计划而非执行）。

```xml
<tool_persistence>
- Use tools whenever they improve correctness
- Do not stop early when another tool call would improve the result
- Keep calling tools until: task complete AND verified
</tool_persistence>

<prerequisite_checks>
- Check whether prerequisite discovery steps are needed
- Do not skip prerequisite steps
</prerequisite_checks>

<verification>
- Correctness: does output satisfy every requirement?
- Grounding: are factual claims backed by tool outputs?
- Formatting: does output match requested format?
- Safety: confirm scope before executing side effects
</verification>
```

### Google 模型操作指导

- 始终使用绝对路径
- 先验证再修改（read_file/search_files）
- 依赖检查（不假设库可用）
- 并行工具调用
- 非交互式命令（-y, --yes 标志）

## 上下文文件注入

**两条独立加载路径**：

| 文件 | 位置 | 搜索范围 |
|------|------|---------|
| **SOUL.md** | `~/.hermes/SOUL.md` | 全局唯一路径，独立加载（Agent 身份槽） |
| **.hermes.md** | cwd 向上到 git root | 项目级配置，优先级 1 |
| **AGENTS.md** | 仅 cwd | 代码库开发指南，优先级 2 |
| **CLAUDE.md** | 仅 cwd | 兼容 Anthropic 格式，优先级 3 |
| **.cursorrules** | 仅 cwd | 兼容 Cursor 格式，优先级 4 |

**项目上下文文件是互斥的**（first match wins）——找到第一个就停，后面的不加载。SOUL.md 不参与这个竞争，它始终与项目文件共存。

**控制加载的方式**：
- `TERMINAL_CWD` — Gateway 模式下，决定在哪个目录查找项目文件。默认 `Path.home()`
- `skip_context_files=True` — 完全跳过项目文件加载（子 Agent 常用）
- `build_context_files_prompt(skip_soul=True)` — 跳过 SOUL.md（已作为身份槽加载时使用）

扫描注入内容的安全威胁：
```python
# agent/prompt_builder.py:36
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'disregard\s+(your|all|any)\s+(instructions|rules|guidelines)', "disregard_rules"),
    (r'<!--[^>]*(?:ignore|override|system|secret|hidden)[^>]*-->', "html_comment_injection"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials|\.netrc|\.pgpass)', "read_secrets"),
    # ... 共 10 条
]
```

此外还扫描不可见 Unicode（零宽字符、双向控制符等，`_CONTEXT_INVISIBLE_CHARS`）。

## 技能索引注入

技能索引是**系统提示的一部分**，由 `build_system_prompt_parts()` 调用 `build_skills_system_prompt()` 拼入 `stable` 层，与身份、工具指导、平台提示等一起组成完整系统提示。

系统提示在会话内只构建一次（缓存在 `self._cached_system_prompt`），仅在上下文压缩后才重建，保证每轮对话复用同一份 → LLM prefix cache 命中率最大化。

## Skills Prompt 两层缓存

构建技能索引文本需要扫描 `~/.hermes/skills/` 下所有文件并解析 frontmatter，为避免重复文件 I/O，使用两层缓存加速：

| 层级 | 存储 | 命中条件 | 失效条件 |
|------|------|----------|----------|
| Layer 1 | 内存 LRU（`OrderedDict`，最多 8 条） | cache_key 匹配（skills_dir + tools + toolsets + platform） | 进程重启 |
| Layer 2 | 磁盘快照（`.skills_prompt_snapshot.json`） | mtime + size manifest 校验通过 | 技能文件变更 |

两层都未命中时，执行全量文件系统扫描 → 写回磁盘快照 + 写入内存缓存。

**注意区分**：此缓存优化的是"生成索引文本"的 I/O 速度，与 LLM API 层的 prefix cache（复用已计算的 token）是不同层面。

## Per-turn 文件修改验证 footer（v0.12.0+，PR #24498，c594a23）

每一轮 agent 调用结束时，Hermes 给 user 反方向附一段 verifier footer，列出**本轮内 write_file / patch 失败但 agent 没自己修好的文件**。模型由此能在下一轮立刻看到自己留的烂摊子，不会"忘记修"。

```python
# run_agent.py:5331-5395
def _record_file_mutation_result(self, tool_name, args, result, is_error):
    """write_file / patch 的成功/失败状态写入 self._turn_failed_file_mutations。

    - tool_name 不在 FILE_MUTATING_TOOL_NAMES → 跳过（统一从 agent/tool_result_classification 导入，c3094b4）
    - 失败 → 第一次失败的 error preview 存进 {path: {error_preview, tool}}
    - 同一 path 后续成功 → state.pop(path) 清掉
    - 同一 path 后续不同错误 → 不覆盖第一次的 error（避免噪声）
    """
```

**判定"成功"** 走 `file_mutation_result_landed(tool_name, result)`——并非靠 `is_error` flag，而是看 tool 结果里的 landed 状态。`fix(da0ddbf)` 起，landed mutation 即使 tool 报错也算成功（compaction 跑 patch 时 tool result 拿不到，但变更已落盘）。

**开关**：

| 来源 | Key | 默认 |
|------|-----|------|
| Config | `display.file_mutation_verifier` (bool) | `True` |
| Env | `HERMES_FILE_MUTATION_VERIFIER` (0/1/false/true/no/off) | — |

env > config > 默认。env 不接受时取 config，config 没设时默认 on。

**与 LSP 协同**（PR #24168，83b9389）：`write_file` / `patch` 落盘后跑 LSP 语义诊断（pyright / tsc / shellcheck），把 diagnostics 作为单独 channel 返回给 agent（不是错误）。配合 in-process linter（PR #20191，5168226，覆盖 JSON / YAML / TOML / Python）形成两层 post-write 检查：
1. **delta lint**（in-process）：写完立刻 lint，差量 diagnostic 直接附在 tool result 里
2. **LSP semantic**（external）：跑真正的 language server，结果作为 `lsp_diagnostics` 字段

## 角色切换

某些模型使用 `developer` 角色而不是 `system` 角色：

```python
# agent/prompt_builder.py:418
DEVELOPER_ROLE_MODELS = ("gpt-5", "codex")
```

实际切换发生在传输层 `agent/transports/chat_completions.py`（约第 231 / 410 行）：当模型名匹配 `DEVELOPER_ROLE_MODELS` 时，把首条消息的 `role` 从 `system` 改写为 `developer`。

## 相关页面

- [[aiagent-class]] — AIAgent 类实体页（构造函数与方法）
- [[prompt-builder-architecture]] — 系统提示模块化构建架构
- [[context-compressor-architecture]] — 上下文压缩与摘要机制

## 相关文件

- `run_agent.py` — `AIAgent` 类定义与转发器方法（4123 行）
- `agent/agent_init.py` — `init_agent()`，构造函数实现（1504 行）
- `agent/conversation_loop.py` — `run_conversation()`，对话循环（4099 行）
- `agent/system_prompt.py` — 三层系统提示组装（346 行）
- `agent/prompt_builder.py` — 提示文本常量、上下文文件加载、技能索引（1465 行）
- `agent/tool_executor.py` — 工具调用执行（顺序/并发，910 行）
- `agent/codex_runtime.py` — Codex app-server / Responses 运行时（448 行）
- `agent/conversation_compression.py` — 会话压缩接入（592 行）
- `agent/context_compressor.py` — 上下文压缩核心
- `agent/agent_runtime_helpers.py` — 消息清洗等运行时辅助（2158 行）
- `agent/prompt_caching.py` — Anthropic prompt 缓存
- `model_tools.py` — 工具编排
