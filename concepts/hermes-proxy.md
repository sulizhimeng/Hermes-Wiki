---
title: Hermes Proxy (OAuth → OpenAI 兼容本地代理)
created: 2026-05-22
type: concept
tags: [proxy, oauth, openai-compatible, adapter, credential-forwarding]
sources: [hermes_cli/proxy/__init__.py, hermes_cli/proxy/cli.py, hermes_cli/proxy/server.py, hermes_cli/proxy/adapters/base.py, hermes_cli/proxy/adapters/__init__.py, hermes_cli/proxy/adapters/nous_portal.py, hermes_cli/proxy/adapters/xai.py]
---

# Hermes Proxy

## 概述

`hermes proxy` 把 OAuth 订阅（Nous Portal、xAI SuperGrok，未来扩展 Claude Pro / ChatGPT Pro）**暴露成 OpenAI-compatible 本地 HTTP 端点**。任何认 OpenAI API 的客户端（Codex CLI、Aider、Cline、Continue、自家脚本）都能直接接，无需 API key，复用你已有的订阅。

v0.14 落地（PR `#25969`）。代码在 `hermes_cli/proxy/`。

### 设计原则

> The proxy is intentionally minimal. It does NOT mediate, log, transform, or rewrite request/response bodies—it's a credential-attaching forwarder.
> —— `hermes_cli/proxy/server.py:1-10`

剥离客户端 Authorization header → 注入上游 OAuth bearer → 透传 SSE/JSON 原样。无任何 prompt 改写、计费拦截或 logging。

---

## 启动

```bash
hermes proxy start [--provider nous|xai] [--host 127.0.0.1] [--port 8645]
```

| 参数 | 默认 | 说明 |
|------|------|------|
| `--provider` | `nous` | 上游 adapter 名 |
| `--host` | `127.0.0.1` | 绑定地址，`0.0.0.0` 开放局域网 |
| `--port` | `8645` | 端口 |

入口：`hermes_cli/proxy/cli.py:30-75` `cmd_proxy_start()`：

1. 检查 aiohttp 可用（line 35），缺则提示 `hermes-agent[messaging]`
2. 按 provider 名解析 adapter（line 41）
3. 校验已登录 `adapter.is_authenticated()`（line 46）
4. `asyncio.run(run_server(adapter, host=host, port=port))`（line 69）

服务器启动：`server.py:255-300` `run_server()` 起 aiohttp TCP site + 信号 handler。

### 状态查看

```bash
hermes proxy status
hermes proxy providers   # 或 proxy list
```

`status` 输出 adapter 就绪情况 + token 过期时间。

---

## Adapter 抽象

### `UpstreamCredential`（`adapters/base.py:21-36`）

```python
@dataclass(frozen=True)
class UpstreamCredential:
    bearer: str                  # token 本体，无 "Bearer " 前缀
    base_url: str                # 如 https://inference-api.nousresearch.com/v1
    token_type: str = "Bearer"
    expires_at: Optional[str] = None
```

### `UpstreamAdapter` ABC（`adapters/base.py:38-107`）

| 方法 | 用途 |
|------|------|
| `name` (property) | provider 键，如 `"nous"` |
| `display_name` (property) | 日志名 |
| `allowed_paths` (property) | `FrozenSet[str]` 白名单路径 |
| `is_authenticated()` | 快速非阻塞检查 |
| `get_credential()` | 返回 fresh bearer + base_url（必要时刷新） |
| `get_retry_credential()` | 上游 401 后给替代凭据（可选） |

### Registry（`adapters/__init__.py:16-19`）

```python
ADAPTERS: Dict[str, Type[UpstreamAdapter]] = {
    "nous": NousPortalAdapter,
    "xai": XAIGrokAdapter,
}
```

`get_adapter(name) → UpstreamAdapter` 工厂在 line 22。

---

## 已支持的 Provider

### Nous Portal（`nous`）

`NousPortalAdapter` 类在 `adapters/nous_portal.py:50`，`allowed_paths` property 在 line 67 返回 module-level `_ALLOWED_PATHS`（line 40-47）。

**白名单**：
```python
{"/chat/completions", "/completions", "/embeddings", "/models"}
```

**OAuth 集成**：

- 读 `~/.hermes/auth.json` 状态
- 调 `resolve_nous_runtime_credentials()` 刷新 access token + 拿/换 `agent_key`
- 双模式：auto + legacy session key；401 时自动用 legacy bearer 重试
- base URL：`DEFAULT_NOUS_INFERENCE_URL`（典型 `https://inference-api.nousresearch.com/v1`，line 140）

### xAI Grok OAuth（`xai`）

`XAIGrokAdapter` 类在 `adapters/xai.py:31`，`allowed_paths` property 在 line 49 返回 module-level `_ALLOWED_PATHS`（line 20-28）。

**白名单**：
```python
{"/responses", "/chat/completions", "/completions", "/embeddings", "/models"}
```
比 Nous 多 `/responses` 端点。

**OAuth 集成**：

- `CredentialPool` 凭据池（`load_pool("xai-oauth")`，line 104）
- `pool.select()` 选一个可用凭据（line 65）
- 401 时：先 refresh，失败则 mark exhausted 并轮换（line 90-92）

**base URL**：凭据条目的 `runtime_base_url` 或 `base_url`，默认 `DEFAULT_XAI_OAUTH_BASE_URL`（line 122-127）

**Auth 提示**：`"hermes auth add xai-oauth --type oauth"`（line 34）

---

## 路由形状

所有请求挂在 `/v1/<path>`：

```python
app.router.add_route("*", "/v1/{tail:.*}", handle_proxy)
```
位置：`server.py:250`

例如 nous adapter 在 `127.0.0.1:8645`：

- `POST /v1/chat/completions`
- `POST /v1/completions`
- `POST /v1/embeddings`
- `GET /v1/models`

健康检查（不转发）：

- `GET /health` → JSON status + 上游名（line 99-106）

**路径校验**：`rel_path not in adapter.allowed_paths` → 返回 404 `path_not_allowed`（line 124-131）

---

## 请求转发

`handle_proxy()` 在 `server.py:119-246`：

1. 提取相对路径
2. 验证路径白名单
3. `adapter.get_credential()` 拿 bearer + base_url；失败 → 401 `upstream_auth_failed`（line 134-137）
4. 读 body 进内存（line 143）
5. **过滤 hop-by-hop header**（line 153），见下
6. 注入 `Authorization: Bearer <cred.bearer>`（line 154）
7. 拼上游 URL：`<base_url><rel_path>?<query_string>`（line 148-151）
8. aiohttp 发：connect timeout 15s，read timeout 300s（line 145）

### Hop-by-hop header 过滤（`server.py:36-50`）

```python
_HOP_BY_HOP_HEADERS = frozenset({
    "host", "content-length", "connection", "keep-alive",
    "proxy-authenticate", "proxy-authorization", "te", "trailers",
    "transfer-encoding", "upgrade", "authorization",
})
```

请求 header 在 line 153 过滤；响应 header 在 line 230 过滤（保留 Content-Encoding / Content-Length，由 aiohttp 重算）。

### 401 重试（`server.py:209-225`）

上游回 401：

1. `adapter.get_retry_credential(failed_credential=cred, status_code=401)`
2. Nous：JWT 失败时用 legacy session key 重试
3. xAI：pool refresh，失败则轮换到下一个可用凭据
4. 拿到新凭据 → 重发请求

---

## 流式响应

完全原样转发：

1. 起 `web.StreamResponse`，承袭上游 status + 过滤后 headers（line 228-231）
2. `async for chunk in upstream_resp.content.iter_any():` chunked 读（line 235）
3. 直接写回 client（line 237）
4. SSE 帧逐字保留：

```
data: {"choices":[{"delta":{"content":"Hello"}}]}

data: {"choices":[{"delta":{"content":" world"}}]}

data: [DONE]
```

---

## 错误响应

OpenAI 风格 JSON（`_json_error()` line 56）：

```json
{
  "error": {
    "message": "...",
    "type": "code_string",
    "code": "code_string"
  }
}
```

| code | HTTP | 触发 |
|------|------|------|
| `upstream_auth_failed` | 401 | adapter 拿不到凭据 |
| `path_not_allowed` | 404 | 路径不在 allowed_paths |
| `upstream_unreachable` | 502 | 连不上上游 |
| `upstream_timeout` | 504 | 上游超时 |

---

## 并发安全

adapter 内用 thread lock 串行化凭据解析：

- `NousPortalAdapter._lock` 在 `_get_credential()`（line 102）
- `XAIGrokAdapter._lock` 在 `get_credential()`（line 57）

跨进程的 token 刷新和持久化由各自的 auth 子系统（Nous 共享 runtime resolver、xAI credential pool）负责。

---

## 客户端配置示例

任意 OpenAI 兼容客户端：

```bash
export OPENAI_API_KEY=any-string-ignored
export OPENAI_BASE_URL=http://127.0.0.1:8645/v1
```

然后 `aider --model claude-opus-4-7`、`cline`、`codex` 等直接用，credential 由 proxy 注入。

---

## 文件索引

| 功能 | 文件 : 行 |
|------|-----------|
| 模块文档串 + 导出 | `__init__.py:1-21` |
| CLI `proxy start` | `cli.py:30` |
| CLI `proxy status` | `cli.py:78` |
| `create_app()` aiohttp factory | `server.py:85` |
| `handle_proxy()` | `server.py:119` |
| `_filter_request_headers` / `_filter_response_headers` | `server.py:62` / `:72` |
| `UpstreamCredential` | `adapters/base.py:21` |
| `UpstreamAdapter` ABC | `adapters/base.py:38` |
| Registry + `get_adapter()` | `adapters/__init__.py:16` |
| `NousPortalAdapter` 类 | `adapters/nous_portal.py:50` |
| `_ALLOWED_PATHS` (Nous) | `adapters/nous_portal.py:40` |
| `XAIGrokAdapter` 类 | `adapters/xai.py:31` |
| `_ALLOWED_PATHS` (xAI) | `adapters/xai.py:20` |

> Module-level constants: `hermes_cli/goals.py` 762 lines, `kanban_db.py` 6286 lines, `agent/curator.py` 1781 lines, `proxy/server.py` 308 lines, `proxy/adapters/base.py` 109 lines（截至 HEAD `09afafb87`）。

---

## 不做的事

- 不缓存
- 不日志请求 body
- 不改 prompt / tools
- 不计费
- 不限流（除上游本身）
- 不做 token 翻译

任何上述需求都该往上层做，或换专用 LLM 网关产品（如 LiteLLM）。Hermes proxy 的存在意义是**让 OAuth 订阅"伪装"成 API key**，仅此而已。

---

*Last verified: 2026-05-22, HEAD `09afafb87`*
