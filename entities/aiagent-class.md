---
title: AIAgent Class
created: 2026-04-07
updated: 2026-05-19
type: entity
tags: [component, agent, module]
sources: [hermes-agent 源码分析 2026-05-19 (v0.14.0)]
---

# AIAgent Class

## 位置

`run_agent.py:326`（类定义）

## 概述

AIAgent 是 Hermes Agent 的核心对话循环类，负责管理 LLM 交互、工具调用和会话状态。v0.14.0 起 `run_agent.py` 经历大规模重构：类**仍保留在 `run_agent.py`**，但 `__init__`、`run_conversation`、`_build_system_prompt`、工具执行、Codex 运行时等核心实现均已抽取到 `agent/` 子模块，类内只保留**薄转发器（forwarder）方法**。

`run_agent.py` 经过重构：约 12 个辅助模块被抽取到 `agent/` 包中，AIAgent 上的许多方法现在只是薄转发器（thin forwarder），真正的实现位于 `agent/` 下的模块里。

## 构造函数

`__init__`（`run_agent.py:349`）本身是转发器，把所有参数原样传给 `agent/agent_init.py` 的 `init_agent()`。

```python
class AIAgent:
    def __init__(self,
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        api_mode: str = None,           # 例如 "codex_app_server"
        model: str = "",                # 默认空字符串，运行时解析为 "anthropic/claude-opus-4.6"
        max_iterations: int = 90,
        tool_delay: float = 1.0,
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
        fallback_model: dict = None,
        credential_pool=None,
        checkpoints_enabled: bool = False,
        pass_session_id: bool = False,
        # ... 共 60+ 参数：routing params、十余个 callbacks 等
    ):
        """Forwarder — see agent.agent_init.init_agent"""
        from agent.agent_init import init_agent
        init_agent(self, ...)
```

## 核心方法

类内方法多为转发器，把实现委托给 `agent/` 子模块：

| 方法 | 位置 | 转发目标 |
|------|------|----------|
| `chat(message, stream_callback=None) -> str` | `run_agent.py:3880` | 内部调用 `run_conversation`，返回 `final_response` |
| `run_conversation(...)` | `run_agent.py:3867` | `agent/conversation_loop.py:187` `run_conversation()` |
| `_build_system_prompt(...)` | `run_agent.py:2164` | `agent/system_prompt.py` `build_system_prompt()` |
| `_build_system_prompt_parts(...)` | `run_agent.py:2159` | `agent/system_prompt.py` `build_system_prompt_parts()` |
| `_run_codex_app_server_turn(...)` | `run_agent.py:3894` | `agent/codex_runtime.py` `run_codex_app_server_turn()` |
| `_execute_tool_calls(...)` | `run_agent.py` | `agent/tool_executor.py` `execute_tool_calls_sequential/concurrent` |
| `_sanitize_api_messages(...)` | `run_agent.py:2196` | `agent/agent_runtime_helpers.py` `sanitize_api_messages()` |

### `chat(self, message: str, stream_callback: Optional[callable] = None) -> str`

简单接口，返回最终响应字符串。

### `run_conversation(self, user_message: str, system_message: str = None, conversation_history: List[Dict] = None, task_id: str = None, stream_callback: Optional[callable] = None, persist_user_message: Optional[str] = None) -> Dict[str, Any]`

完整接口，返回 `{final_response, messages}` 字典。实际循环逻辑在 `agent/conversation_loop.py`。

`AIAgent.run_conversation` 本身只是 `run_agent.py:3859` 的转发器，真正的实现是 `agent/conversation_loop.py:187` 的 `run_conversation`（带一个前导 `agent` 参数）。`_build_system_prompt`（`run_agent.py:2156`）同样是转发器 → `agent/system_prompt.py` 的 `build_system_prompt`。

## 对话循环

实现位于 `agent/conversation_loop.py:187`：

```python
agent.iteration_budget = IterationBudget(agent.max_iterations)
api_call_count = 0

# 可选的 codex_app_server 运行时：整轮交给 codex app-server 子进程
if agent.api_mode == "codex_app_server":
    return agent._run_codex_app_server_turn(...)

while (api_call_count < agent.max_iterations
       and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
    api_call_count += 1
    # ... 发起 API 调用
    if assistant_message.tool_calls:
        agent._execute_tool_calls(assistant_message, messages,
                                  effective_task_id, api_call_count)  # → agent/tool_executor.py
    else:
        return final_response
```

## 关键特性

- **完全同步** — 不使用 asyncio
- **转发器架构** — 类体精简，实现分散到 `agent/` 子模块
- **工具循环** — 支持多轮工具调用，顺序/并发两条执行路径
- **迭代预算** — `IterationBudget` 控制最大 API 调用次数
- **平台感知** — 根据平台注入不同提示
- **记忆集成** — 自动加载和注入记忆
- **技能集成** — 构建技能索引（带两层缓存）
- **上下文压缩** — 自动管理上下文长度
- **Codex 运行时** — 支持 `codex_app_server` / `codex_responses` api_mode
- **系统提示持久化** — system prompt 存入 SessionDB，Gateway 每轮新建 AIAgent 时回读以复用前缀缓存

## 相关页面

- [[agent-loop-and-prompt-assembly]] — Agent 核心循环与系统提示组装
- [[multi-agent-architecture]] — 子代理委派与迭代预算系统
- [[prompt-builder-architecture]] — 系统提示构建架构

## 相关文件

- `run_agent.py` — `AIAgent` 类定义与转发器方法（4123 行）
- `agent/agent_init.py` — `init_agent()`，构造函数实现（1504 行）
- `agent/conversation_loop.py` — `run_conversation()`，对话循环（4099 行）
- `agent/system_prompt.py` — 系统提示三层组装（346 行）
- `agent/tool_executor.py` — 工具调用执行（910 行）
- `agent/codex_runtime.py` — Codex app-server / Responses 运行时（448 行）
- `agent/agent_runtime_helpers.py` — 消息清洗等运行时辅助（2158 行）
- `agent/prompt_builder.py` — 系统提示文本常量与上下文文件加载（1465 行）
- `model_tools.py` — 工具编排
