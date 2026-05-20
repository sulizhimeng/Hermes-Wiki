---
title: hermes proxy —— OAuth Provider 的 OpenAI 兼容本地代理
created: 2026-05-20
updated: 2026-05-20
type: concept
tags: [proxy, oauth, openai-compat, integration]
source_files:
  - hermes_cli/proxy/server.py
  - hermes_cli/proxy/cli.py
  - hermes_cli/proxy/adapters/base.py
  - hermes_cli/proxy/adapters/nous_portal.py
  - hermes_cli/proxy/adapters/xai.py
verified_against: hermes-agent HEAD (2026-05-20)
---

# `hermes proxy` —— OAuth Provider 的 OpenAI 兼容本地代理（v0.14.0）

```
┌────────────────┐  HTTP    ┌────────────────────────────┐  HTTPS   ┌──────────────────┐
│  Codex CLI     │ ───────► │  hermes proxy              │ ───────► │ Nous Portal /    │
│  Aider         │          │  http://127.0.0.1:8888     │          │ xAI Grok         │
│  Cline         │          │                            │          │                  │
│  Continue      │ ◄─────── │  (替换 Authorization 头)   │ ◄─────── │ (SSE 流式)       │
│  你的脚本      │   SSE    │                            │   SSE    │                  │
└────────────────┘          └────────────────────────────┘          └──────────────────┘
```

启动一个本地 HTTP server，听 `http://<host>:<port>/v1/<path>`，把请求**仅替换 `Authorization` 头**后透传给上游。客户端用什么 API key 都无所谓 —— hermes 现取现塞它管理的 OAuth bearer。

效果：任何 OpenAI-compatible 工具都能用 Claude Pro / ChatGPT Pro / SuperGrok 订阅作后端，**不需要 API key**。

---

## 1. 命令

```
hermes proxy [--provider nous|xai] [--host 127.0.0.1] [--port 8888]
```

默认 `nous`（Nous Portal）。Provider 列表来自 `hermes_cli/proxy/adapters/__init__.py:ADAPTERS`。

---

## 2. 工作流

源码：`hermes_cli/proxy/server.py:1-50`。

1. 客户端发 `POST http://localhost:8888/v1/chat/completions` 或 `/v1/responses` ...
2. server 调 `adapter.resolve()` 拿当下有效的 `UpstreamCredential`（含 bearer + base_url）。
3. server **重建请求头**：
   - 删 hop-by-hop headers（`_HOP_BY_HOP_HEADERS` frozenset 含 `host`, `content-length`, `connection`, `keep-alive`, `proxy-authenticate`, `proxy-authorization`, `te`, `trailers`, `transfer-encoding`, `upgrade`, `authorization`）。
   - 加 `Authorization: Bearer <hermes-managed-token>`。
4. server 透传 body（不解析、不变换、不记录）到 `<upstream-base-url>/<path>`。
5. response 以 raw bytes 流回客户端，**SSE 保持原样**。

明确**不做**的事：

- 不 mediate
- 不 log（除 INFO 摘要）
- 不 transform request/response body
- 不 rewrite SSE 事件

就是一个**凭证-attach 转发器**。

---

## 3. Adapter 协议

源码：`hermes_cli/proxy/adapters/base.py`，109 行。

```python
@dataclass
class UpstreamCredential:
    base_url: str
    bearer: str
    allowed_paths: FrozenSet[str]

class UpstreamAdapter(ABC):
    auth_hint: str = ""           # 配置失败时给用户的提示

    @abstractmethod
    async def resolve(self) -> UpstreamCredential:
        """返回当下有效的凭证 + 上游 base_url + 允许的路径白名单"""
```

每次客户端请求，server 都重新 `await adapter.resolve()` —— 因此 OAuth refresh 在请求路径上**自动**触发，不需要后台线程。

---

## 4. 内置 Adapter

### 4.1 Nous Portal（`nous_portal.py`）

源码：`hermes_cli/proxy/adapters/nous_portal.py`，191 行。

- 读 `~/.hermes/auth.json`（与 hermes 主程序共享）。
- 用 `hermes_cli.auth.resolve_nous_runtime_credentials()` 解出 bearer：
  - 兼容 NAS invoke JWT 和 legacy opaque session key（`agent_key` 字段两种形式）。
  - 刷新窗口内自动 refresh access token。
- `base_url` = `DEFAULT_NOUS_INFERENCE_URL`（可被 `auth.json` 覆盖）。
- **凭证终端态保护**：refresh 永久失败时（用户撤销 / 账户停用）调用 `_quarantine_nous_oauth_state` 把状态隔离，避免循环抓 500。
- 允许路径白名单：`/chat/completions`, `/responses`, `/completions`, `/embeddings`, `/models` —— 其余 404。

### 4.2 xAI Grok（`xai.py`）

源码：`hermes_cli/proxy/adapters/xai.py`，136 行。

- 走 `agent.credential_pool.CredentialPool`（参见 [[credential-pool-and-isolation]]）的 `xai-oauth` 提供方 entry。
- `base_url` = `DEFAULT_XAI_OAUTH_BASE_URL`（可被 entry 覆盖）。
- `auth_hint = "hermes auth add xai-oauth --type oauth"`。
- 允许路径：`/responses`, `/chat/completions`, `/completions`, `/embeddings`, `/models`。

---

## 5. 为什么有白名单

源码注释（`xai.py` 顶部）：

> xAI's public API is OpenAI-compatible for the endpoints Hermes commonly uses. The Responses endpoint is included because Hermes' native xAI runtime uses `codex_responses` mode.

但 OAuth token 通常有**比 API key 更宽**的权限（比如能查账户、订阅状态）。白名单避免了"hermes proxy 不小心成了万能 OAuth 透传"，限制流量到合规的 inference API 子集。

任何不在白名单的路径：server 返回 **404**，不向上游发请求。

---

## 6. 与既有体系的关系

| 场景 | 用 hermes proxy | 用 hermes 主程序 |
|------|----------------|-----------------|
| 想用 Codex CLI + Claude Pro 订阅 | ✓ | ✗（Codex 不调用 hermes） |
| 想用 Aider + ChatGPT Pro | ✓ | ✗ |
| 想跟 Cline / Continue 集成 | ✓ | hermes 是 ACP server，但 Cline 不一定支持 |
| 想自己写脚本走 OpenAI SDK | ✓（指 base_url 到本地） | — |
| 想用 hermes 自己的 chat | ✗ | ✓ |

> 这是补**生态兼容**的洞 —— hermes 本身的 chat / TUI / gateway 已经直接用 OAuth bearer 调上游，proxy 只是把同一个能力暴露给**所有 OpenAI-客户端**。

---

## 7. 安全考量

- proxy **只监听** `127.0.0.1` 默认 host —— 不监听 `0.0.0.0`。要远程调请显式过桥（SSH tunnel / Tailscale）。
- 客户端发的 `Authorization` 头**永远被丢弃**，永远被 hermes 管理的 bearer 替换。客户端 API key 完全没用 —— 即使有人尝试用 proxy 当 "auth-bypass" 攻击，bearer 来源仍然是 `~/.hermes/auth.json` 这个本机文件。
- aiohttp 依赖 lazy 检测：`from .server import AIOHTTP_AVAILABLE`，缺时 `cmd_proxy_start` 返回非零 + 提示 `pip install 'hermes-agent[messaging]'`。

---

## 8. 不变量

- 一次请求 = 一次 `adapter.resolve()`（OAuth refresh 在请求路径上）。
- request body **不解析，不变换** —— 二进制透传。
- SSE 流**原样转发**（不缓冲整个 response）。
- 路径白名单**写死在 adapter**，违例返 404。
- hop-by-hop 头一律去除，重建 `Authorization`。

---

## 9. 验证

```
hermes_cli/proxy/server.py:1-50          docstring 明确"凭证 attaching forwarder"
hermes_cli/proxy/server.py:_HOP_BY_HOP_HEADERS  frozenset 11 项
hermes_cli/proxy/cli.py:cmd_proxy_start  aiohttp lazy check + 报错文案
hermes_cli/proxy/adapters/base.py:1-109  UpstreamAdapter ABC
hermes_cli/proxy/adapters/nous_portal.py:33-50  quarantine on terminal refresh failure
hermes_cli/proxy/adapters/xai.py:_ALLOWED_PATHS  5 个路径
```
