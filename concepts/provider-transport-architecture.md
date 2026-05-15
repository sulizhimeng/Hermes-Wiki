---
title: Provider Transport 架构
created: 2026-04-18
updated: 2026-05-15
type: concept
tags: [architecture, module, provider, transport, api-dispatch]
sources: [agent/transports/base.py, agent/transports/anthropic.py, agent/transports/chat_completions.py, agent/transports/bedrock.py, agent/transports/codex.py, agent/transports/codex_app_server.py, agent/transports/types.py, agent/transports/__init__.py, run_agent.py]
---

# Provider Transport — API 路径统一抽象

## 概述

Provider Transport 是 **v2026.4.17+** 引入的架构级重构，用统一的 ABC 抽象了所有 provider 的 API 数据路径（Anthropic Messages、OpenAI Chat Completions、OpenAI Responses API、AWS Bedrock）。位于 `agent/transports/`（1217 行），替代了之前散落在 `run_agent.py` 各处的 `if api_mode == "anthropic_messages": ... elif ...` 分支判断。

**核心理念**：**一个 provider 的消息转换、工具转换、参数构建、响应规范化，应该聚合在一个类里，而不是散落在调用点。**

## 架构原理

### 四个抽象方法 + 三个可选钩子

```python
# agent/transports/base.py
class ProviderTransport(ABC):
    @property
    @abstractmethod
    def api_mode(self) -> str:
        """处理的 api_mode 字符串（如 'anthropic_messages'）"""

    @abstractmethod
    def convert_messages(self, messages, **kwargs) -> Any:
        """OpenAI 格式消息 → provider 原生格式"""

    @abstractmethod
    def convert_tools(self, tools) -> Any:
        """OpenAI 工具定义 → provider 原生格式"""

    @abstractmethod
    def build_kwargs(self, model, messages, tools=None, **params) -> Dict:
        """组装完整的 API 调用 kwargs（通常内部调用前两个方法）"""

    @abstractmethod
    def normalize_response(self, response, **kwargs) -> NormalizedResponse:
        """原始响应 → 共享的 NormalizedResponse 类型（唯一返回 transport 层类型的方法）"""

    # ── 可选钩子 ───────────────────────────────────────────
    def validate_response(self, response) -> bool: ...       # 结构校验
    def extract_cache_stats(self, response) -> Optional[Dict]: ...  # cache hit/create 提取
    def map_finish_reason(self, raw_reason) -> str: ...      # stop reason 映射
```

**设计要点**：
- Transport **只负责数据路径**，不管 client 生命周期、streaming、auth、credential refresh、retry、interrupt handling——这些都在 `AIAgent` 上
- `normalize_response` 是唯一返回 transport 层类型（`NormalizedResponse`）的方法，其他方法返回 provider 原生结构

### 已实现的 Transport

| Transport | 文件 | 行数 | api_mode | 覆盖 |
|-----------|------|------|----------|------|
| `AnthropicTransport` | `transports/anthropic.py` | 179 | `anthropic_messages` | Claude（直连、Nous Portal） |
| `ChatCompletionsTransport` | `transports/chat_completions.py` | 614 | `chat_completions` | OpenAI、OpenRouter、Gemini、xAI、custom OpenAI 兼容 |
| `ResponsesApiTransport` | `transports/codex.py` | 270 | `codex_responses` | OpenAI Codex、xAI Grok OAuth（Responses API） |
| `BedrockTransport` | `transports/bedrock.py` | 154 | `bedrock_converse` | AWS Bedrock（Converse API） |
| `NormalizedResponse` | `transports/types.py` | 162 | — | 共享响应类型 |
| 基类 + 注册表 | `transports/base.py` + `__init__.py` | 89 + 68 | — | ABC + `get_transport()` 惰性发现 |

> **api_mode 命名**：注册表实际注册的字符串是 `anthropic_messages`、`chat_completions`、`codex_responses`、`bedrock_converse`（见各 transport 模块尾部的 `register_transport(...)` 调用）。OpenAI Responses API 走的 api_mode 是 `codex_responses`。

### 注册表：惰性发现

```python
# agent/transports/__init__.py
def get_transport(api_mode: str) -> ProviderTransport:
    """按需 import 对应的 transport 模块，触发模块级 register_transport() 调用"""
    ...

def register_transport(api_mode: str, transport_cls: type) -> None:
    """transport 模块在 import 时调用，把自己注册到 registry"""
    ...
```

首次 `get_transport("anthropic_messages")` 调用时才 import `transports/anthropic.py`——**延迟到实际使用**，启动不会因为 import 一堆 SDK 而变慢。

## Codex App-Server 运行时（可选 opt-in，#24182）

除上述四个标准 transport，Hermes 还提供一个**可选的替代运行时**：把 OpenAI/Codex 模型的每个回合交给一个 `codex app-server` 子进程处理，而非走 Hermes 自己的工具派发循环。**默认行为不变。**

| 模块 | 文件 | 行数 | 职责 |
|------|------|------|------|
| App-Server 客户端 | `transports/codex_app_server.py` | 368 | stdio 上的 newline-delimited JSON-RPC 2.0 speaker：spawn `codex app-server`、init 握手、请求/响应、通知队列、服务端发起的请求队列（审批往返）、可中断阻塞读 |
| 会话适配器 | `transports/codex_app_server_session.py` | 810 | `CodexAppServerSession`——每个 `AIAgent` 实例一个惰性会话 |
| 事件投影器 | `transports/codex_event_projector.py` | 312 | 把 codex 的 `item/*` 通知转回 Hermes 标准 `{role, content, tool_calls, tool_call_id}` 消息形状，使记忆/技能 review 仍可工作 |

启用方式：
- `_VALID_API_MODES` 新增 `codex_app_server`（`hermes_cli/runtime_provider.py`）
- `_maybe_apply_codex_app_server_runtime()` 在 `_resolve_runtime_from_pool_entry()` 末尾调用——**仅当** config.yaml 中 `model.openai_runtime: codex_app_server` **且** provider 属于 `{openai, openai-codex}` 时才把 api_mode 改写为 `codex_app_server`。其他 provider（anthropic、openrouter 等）不会被重路由
- `AIAgent.run_conversation()` 在 `self.api_mode == "codex_app_server"` 时调用 `_run_codex_app_server_turn()`，委托给 `CodexAppServerSession`

注意 `codex_app_server` 并非通过标准 `register_transport()` 注册的 transport——它是一条独立的运行时路径，由 `AIAgent` 直接分支，与四个标准 transport 平级但走子进程协议。

## 在 run_agent.py 中的接入点

`AnthropicTransport`、`ChatCompletionsTransport`、`BedrockTransport`、`ResponsesApiTransport` 替代了 `run_agent.py` 中 **20+ 个直接调用 provider 适配器函数的位置**：

| 场景 | 新方法 |
|------|--------|
| 主 kwargs 构建（按 api_mode 派发） | `transport.build_kwargs(...)` |
| 记忆 flush（build_kwargs + normalize） | `_tflush.build_kwargs` / `_tfn.normalize_response` |
| 迭代上限摘要 + 重试 | `_tsum.build_kwargs` / `_tsum.normalize_response` |
| 响应结构校验 | `transport.validate_response` |
| finish reason 映射（Anthropic stop_reason → OpenAI） | `transport.map_finish_reason` |
| 截断响应的规范化 | `transport.normalize_response` |
| cache 命中/创建统计提取 | `transport.extract_cache_stats` |
| 主 normalize loop | `transport.normalize_response` |

所有 transport 方法调用路径下的 adapter import 完全收敛到 transport 类内部，`run_agent.py` 本身不再直接 import `anthropic_adapter` 等函数。

**零直接 adapter imports 残留**（指 transport 方法的调用路径）。

辅助客户端（`agent/auxiliary_client.py`）也迁移到 transport（compression、memory flush、session summarization 路径）。

## 设计优越性

### 对比旧架构

| 维度 | 旧方案 | Transport ABC |
|------|--------|---------------|
| 分支代码 | `run_agent.py` 散落 `if api_mode == ...` 判断 | 单点 `get_transport(api_mode)` |
| 添加新 provider | 改多处（转换、normalize、cache stats...） | 新增一个 transport 子类 |
| 测试 | 难以单独测消息/工具转换 | 每个方法可独立单元测 |
| 循环依赖 | 容易 | 零——transport 只 import `base` / `types` |
| 启动开销 | 可能 eager import 所有 SDK | 惰性 import，按需加载 |

### 单一职责

- **Transport**：消息/工具格式转换 + 响应规范化
- **AIAgent**：client 生命周期、streaming、auth、retry、interrupt
- **Adapter**（旧代码）：保留，transport 内部委托给它，逐步废弃

### 迁移状态

| Provider | Transport 覆盖 | 状态 |
|----------|---------------|------|
| Anthropic | AnthropicTransport（委托 `anthropic_adapter.py`） | 全路径完成 |
| Chat Completions（OpenAI 兼容） | ChatCompletionsTransport | 全路径完成 |
| OpenAI Responses API（Codex、xAI Grok OAuth） | ResponsesApiTransport（`codex_responses`） | 全路径完成 |
| AWS Bedrock | BedrockTransport | 全路径完成 |
| Codex App-Server 运行时 | `CodexAppServerSession`（独立路径，opt-in） | 子进程运行时，默认关闭 |
| Auxiliary Client（压缩/记忆） | 已迁移到 Transport | 完成 |

## 与其他系统的关系

- [[auxiliary-client-architecture]] — auxiliary_client 已迁移到 Transport
- [[smart-model-routing]] — transport 基于 api_mode 派发，与模型路由配合
- [[interrupt-and-fault-tolerance]] — 中断、retry 仍在 AIAgent 层，不属于 transport 职责
- [[prompt-caching-optimization]] — cache 统计通过 `extract_cache_stats` 钩子暴露

## 相关文件

- `agent/transports/base.py`（89 行） — `ProviderTransport` ABC
- `agent/transports/types.py`（162 行） — `NormalizedResponse` 共享类型
- `agent/transports/__init__.py`（68 行） — 注册表 + 惰性发现
- `agent/transports/anthropic.py`（179 行） — Anthropic Messages
- `agent/transports/chat_completions.py`（614 行） — Chat Completions
- `agent/transports/codex.py`（270 行） — OpenAI Responses API（`codex_responses`）
- `agent/transports/bedrock.py`（154 行） — AWS Bedrock Converse
- `agent/transports/codex_app_server.py`（368 行） — Codex app-server JSON-RPC 客户端
- `agent/transports/codex_app_server_session.py`（810 行） — Codex 运行时会话适配器
- `agent/transports/codex_event_projector.py`（312 行） — Codex item → Hermes 消息投影
- `run_agent.py` — 10+ 接入点
- `agent/auxiliary_client.py` — 辅助路径已迁移
