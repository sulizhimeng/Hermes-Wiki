---
title: ProviderProfile 插件系统
created: 2026-05-20
updated: 2026-05-20
type: concept
tags: [provider, transport, plugin, model-routing]
source_files:
  - providers/base.py
  - providers/__init__.py
  - plugins/model-providers/
  - plugins/model-providers/README.md
verified_against: hermes-agent HEAD (2026-05-20)
---

# ProviderProfile 插件系统（v0.13.0 起）

Hermes 之前把每个 provider 的"怪癖"散落在 `run_agent.py`、`agent/auxiliary_client.py` 和 `agent/model_metadata.py` 的 if/elif 分支里。v0.13.0 用 `ProviderProfile` ABC + `plugins/model-providers/` 插件目录把它**收敛到声明式数据类**。

这是 [[provider-transport-architecture]] 的姐妹页 ——

- **Transport**：负责数据路径（convert_messages → convert_tools → build_kwargs → normalize_response），少而稳，4 个内置。
- **ProviderProfile**：声明性描述某个 provider 的差异点，多且常增减，**纯数据**。

```
┌────────────────────────────────────────────────────────────┐
│ AIAgent                                                    │
│    │                                                       │
│    │ get_provider_profile("nvidia") → ProviderProfile      │
│    │ get_transport("chat_completions") → Transport         │
│    │                                                       │
│    │ kwargs = transport.build_kwargs(profile, ...)         │
│    │ resp   = transport.send(...)                          │
│    └ resp   = transport.normalize_response(resp, profile)  │
└────────────────────────────────────────────────────────────┘
```

---

## 1. `ProviderProfile` ABC

源码：`providers/base.py:39`。

```python
@dataclass
class ProviderProfile:
    # ── Identity ─────────────────────────────────────────
    name: str
    api_mode: str = "chat_completions"   # 对应 Transport
    aliases: tuple = ()

    # ── Human-readable metadata ───────────────────────────
    display_name: str = ""               # "GMI Cloud"
    description: str = ""                # 子标题
    signup_url: str = ""                 # 注册链接

    # ── Auth & endpoints ─────────────────────────────────
    env_vars: tuple = ()                 # ("XAI_API_KEY", ...)
    base_url: str = ""
    models_url: str = ""                 # 显式覆盖；空则 base_url + "/models"
    auth_type: str = "api_key"
        # api_key | oauth_device_code | oauth_external | copilot | aws_sdk
    supports_health_check: bool = True   # doctor 是否探活

    # ── Model catalog ─────────────────────────────────────
    fallback_models: tuple = ()          # /model picker 兜底（仅 agentic 模型）
    hostname: str = ""                   # URL → provider 反查（model_metadata.py）

    # ── Client-level quirks ───────────────────────────────
    default_headers: dict[str, str]

    # ── Request-level quirks ──────────────────────────────
    fixed_temperature: Any = None        # None=透传, OMIT_TEMPERATURE=不发
    default_max_tokens: int | None = None
    default_aux_model: str = ""          # auxiliary 任务默认模型
```

钩子方法（子类可选覆盖）：

```python
def prepare_messages(self, messages) -> list:
    """codex 字段清洗后 / developer 角色翻译前调用"""
    return messages

def build_extra_body(self, *, session_id=None, **ctx) -> dict:
    """provider 特定的 extra_body"""
    return {}

def build_api_kwargs_extras(self, *, reasoning_config=None, **ctx):
    """返回 (extra_body_additions, top_level_kwargs)
       split 让 OpenRouter (extra_body.reasoning) 和 Kimi (api_kwargs.reasoning_effort)
       这种不同位置的差异由 profile 负责"""
    return {}, {}

def fetch_models(self, *, api_key=None, timeout=8.0) -> list[str] | None:
    """从 provider 的 /models 端点拉真实可用 model 列表，失败返 None"""
    ...
```

**关键设计**：profile 是**纯数据**，**不构造 client，不轮换凭证，不管流式**。所有这些仍住在 `AIAgent`，因此 profile 是无状态可序列化的描述。

---

## 2. 插件目录布局

```
plugins/model-providers/<name>/
├── __init__.py           # 调用 register_provider(profile) 一次
└── plugin.yaml           # 清单：name / kind: model-provider / version / description
```

29 个内置（HEAD 期），按字母序：

```
ai-gateway     alibaba        alibaba-coding-plan  anthropic
arcee          azure-foundry  bedrock              copilot
copilot-acp    custom         deepseek             gemini
gmi            huggingface    kilocode             kimi-coding
minimax        nous           novita               nvidia
ollama-cloud   openai-codex   opencode-zen         openrouter
qwen-oauth     stepfun        xai                  xiaomi          zai
```

### 用户覆盖

```
$HERMES_HOME/plugins/model-providers/<same-name>/
```

**同名时覆盖内置**（`register_provider` 是 last-writer-wins）。允许任意用户 monkey-patch 或替换内置 profile，**不需要 fork 仓库**。

### 向后兼容

`providers/*.py` 单文件 profile 仍被 `pkgutil.iter_modules` 发现，让 editable 安装的旧代码不破。**新写的 profile 应该用插件目录布局**。

---

## 3. 发现机制

源码：`providers/__init__.py`。

```python
def _discover_providers() -> None:
    """两个目录扫一遍：
       1. plugins/model-providers/   （仓库内置）
       2. $HERMES_HOME/plugins/model-providers/  （用户覆盖）

       每个 __init__.py 被 importlib import 一次 → 触发 register_provider()
    """

# 公共 API：
def get_provider_profile(name: str) -> ProviderProfile | None:
    """匹配 name 或 alias，未匹配返 None；不抛"""

def list_providers() -> list[ProviderProfile]:
    """所有已注册"""
```

**懒发现**：第一次 `get_provider_profile` 或 `list_providers` 被调用时才扫，不在 import 时扫 —— 这也是 v0.14.0 冷启动 −19s 的一部分。

---

## 4. 实例：怎么加一个新 Provider

```python
# plugins/model-providers/myprov/__init__.py
from providers import register_provider
from providers.base import ProviderProfile

my_provider = ProviderProfile(
    name="my-provider",
    aliases=("myp", "my"),
    display_name="My Provider",
    description="Cheap multi-model API",
    signup_url="https://my-provider.example.com/keys",
    env_vars=("MYP_API_KEY", "MYP_BASE_URL"),
    base_url="https://api.my-provider.example.com/v1",
    default_aux_model="my-cheap-7b",
)

register_provider(my_provider)
```

```yaml
# plugins/model-providers/myprov/plugin.yaml
name: my-provider
kind: model-provider
version: 1.0.0
description: ...
```

即可：

- `hermes auth add my-provider` 起作用。
- `hermes model` picker 里出现。
- `hermes doctor` 跑 /models 探活。
- model_metadata 反向查 hostname。
- transport 由 `api_mode="chat_completions"` 自动决定。

---

## 5. 与 Transport 的分工

| 维度 | Profile | Transport |
|------|---------|-----------|
| 数量 | 29 内置 + 用户 | 4 内置（anthropic / chat_completions / bedrock / codex 系列） |
| 形态 | dataclass，可序列化 | 类，含 IO |
| 持有 | provider 元信息 | 数据路径方法 |
| 增长频率 | 高（新 provider 频繁） | 低（新 API 协议才增） |
| 关键决策 | `api_mode` 字段 | — |

`api_mode` 字符串是关键的耦合点 ——

```python
profile = get_provider_profile("nvidia")        # api_mode="chat_completions"
transport = get_transport(profile.api_mode)     # ChatCompletionsTransport()
```

未来加新协议（比如 Cohere v2、Mistral le-platforme），只需：

- 加一个 Transport 子类到 `agent/transports/`
- 把 profile 的 `api_mode` 指过去

---

## 6. 与 `agent/model_metadata.py` 的关系

`agent/model_metadata.py` 的 10 级上下文长度解析链（[[smart-model-routing]]）现在**第一步**就是 `get_provider_profile`：

- profile.hostname → URL 反查
- profile.fetch_models() → 真实可用模型列表（命中 models.dev / OpenRouter manifest / Nous Portal manifest 之前作为 short-circuit）

remote model catalog manifest（v0.12.0）通过 profile.fetch_models 的远端 manifest 实现：OpenRouter / Nous Portal 的 fetch_models 现在去拉 manifest，**新模型不发版即可见**。

---

## 7. 不变量

- profile 是**纯数据** —— 不持有 client、不持有 token、不管流式、不管重试。
- `register_provider` last-writer-wins，user > bundled。
- 第一次解析时才扫盘（懒发现）。
- alias 也参与匹配。
- 任意 `api_mode` 字符串都必须对应 `agent/transports/` 里的一个注册项，否则 transport 返回 None，调用方走 legacy 兜底。
- `providers/*.py` 单文件 profile **仍支持**，但优先级低于插件目录。

---

## 8. 验证

```
providers/base.py:39                 class ProviderProfile
providers/base.py:74-80              ProviderProfile.get_hostname()
providers/base.py:90                 prepare_messages
providers/base.py:100                build_extra_body
providers/base.py:113                build_api_kwargs_extras
providers/base.py:130                fetch_models
providers/__init__.py:1-30           设计 docstring
plugins/model-providers/README.md    用户文档（39 行 + 模板）
plugins/model-providers/             29 个目录
agent/transports/__init__.py:51-68   _discover_transports()
```
