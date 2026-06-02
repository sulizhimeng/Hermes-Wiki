---
title: Dashboard OAuth 鉴权闸门（v0.14 增量）
created: 2026-05-27
updated: 2026-06-02
type: concept
tags: [architecture, security, dashboard-auth, oauth, plugins, extensibility, refresh-token]
sources:
  - hermes_cli/dashboard_auth/base.py
  - hermes_cli/dashboard_auth/registry.py
  - hermes_cli/dashboard_auth/middleware.py
  - hermes_cli/dashboard_auth/routes.py
  - hermes_cli/dashboard_auth/cookies.py
  - hermes_cli/dashboard_auth/ws_tickets.py
  - hermes_cli/dashboard_auth/prefix.py
  - hermes_cli/dashboard_auth/audit.py
  - plugins/dashboard_auth/nous/__init__.py
  - hermes_cli/plugins.py
---

# Dashboard OAuth 鉴权闸门（v0.14 增量）

> **NEW 2026-05-27**：dashboard 从只支持 loopback `_SESSION_TOKEN` 单一鉴权扩展为可插拔 OAuth provider 闸门。Phase 0–7 同日合入 master（19 个 `feat(dashboard-auth*)` + 多个 fix/test，commits `8773bbf` → `2fc4615`）。

## 定位与触发条件

Dashboard web server（`hermes dashboard`）原本依靠 loopback 注入的 `_SESSION_TOKEN` 进行同源访问校验，只在 `--insecure` 或绑定 loopback 时可用。本次把它升级为可插拔 OAuth 闸门：

```text
should_require_auth(host, insecure) →
    auth_required: bool  (stash on app.state)
```

- **loopback + 未 --insecure** → `auth_required = False`（保留 `_SESSION_TOKEN` 老路）
- **非 loopback + 未 --insecure** → `auth_required = True`，加载 `AuthGateMiddleware`
- **--insecure** → 永远 False（运维 escape hatch）

`hermes_cli/dashboard_auth/__init__.py`、`hermes_cli/dashboard_auth/middleware.py:32`、判定见 `8773bbf` + `949ad95` 两个 phase-0 commit。

## 模块结构

`hermes_cli/dashboard_auth/`（10 文件、1868 行）：

| 文件 | 行数 | 职责 |
|------|------|------|
| `base.py` | 158 | `DashboardAuthProvider` ABC + 3 类异常 + 2 个 dataclass + `assert_protocol_compliance` |
| `registry.py` | 58 | 进程级线程安全 dict + `register_provider` / `list_providers` / `get_provider` |
| `middleware.py` | 207 | FastAPI 中间件 — 白名单 + cookie 校验 + HTML/JSON 双路 401 |
| `routes.py` | 456 | `/auth/login` / `/auth/callback` / `/auth/logout` / `POST /api/auth/ws-ticket` |
| `cookies.py` | 234 | `session_at` / `session_rt` / PKCE cookie helpers |
| `login_page.py` | 384 | `/login` HTML（Nous DS 风格） |
| `ws_tickets.py` | 87 | 单次性 30s TTL WS-upgrade ticket |
| `prefix.py` | 157 | `X-Forwarded-Prefix` 反向代理 mount |
| `audit.py` | 87 | json-lines 审计日志 |
| `__init__.py` | 40 | 公开 API |

Nous 官方 provider：`plugins/dashboard_auth/nous/__init__.py`（582 行） + `plugin.yaml`。

## Provider 协议（ABC）

`base.py:65-122` 定义 5-方法生命周期：

```python
class DashboardAuthProvider(ABC):
    name: str = ""           # 稳定 lowercase 标识符，永不可变
    display_name: str = ""   # /login 面板用户可见标签

    @abstractmethod
    def start_login(self, *, redirect_uri: str) -> LoginStart:
        """1) 用户点击 'Log in with X'，返回 OAuth 跳转 URL + PKCE/CSRF cookie"""

    @abstractmethod
    def complete_login(self, *, code, state, code_verifier, redirect_uri) -> Session:
        """3) /auth/callback：用 code + verifier 换 Session"""

    @abstractmethod
    def verify_session(self, *, access_token: str) -> Optional[Session]:
        """4) 每请求校验；None = 过期/无效，触发 refresh 或重新登录"""

    @abstractmethod
    def refresh_session(self, *, refresh_token: str) -> Session:
        """5) 即将过期时轮换"""

    @abstractmethod
    def revoke_session(self, *, refresh_token: str) -> None:
        """6) /auth/logout；尽力即可"""
```

异常 → HTTP 状态码映射（middleware 层强制）：

| 异常 | HTTP | 触发 |
|------|------|------|
| `ProviderError` | 503 | IDP 不可达 / 网络故障 |
| `InvalidCodeError` | 400 | callback `code` / `state` 校验失败 |
| `RefreshExpiredError` | 302 → `/login` | refresh token 已死，clear cookie 强制重登 |

### Session dataclass

```python
@dataclass(frozen=True)
class Session:
    user_id: str
    email: str
    display_name: str
    org_id: str          # 无 org 概念的 provider 设空串
    provider: str        # 匹配 registry 的 name
    expires_at: int      # access_token 的 exp claim
    access_token: str    # provider 不透明，Hermes 不解析
    refresh_token: str
```

### LoginStart dataclass

```python
@dataclass(frozen=True)
class LoginStart:
    redirect_url: str           # 浏览器导航到这
    cookie_payload: dict[str, str]  # PKCE state / CSRF nonce
```

`cookie_payload` 中的 cookie 必须 HttpOnly + Secure（HTTPS 时）+ SameSite=Lax + TTL ≤ 10 分钟。

### 协议合规检查

`assert_protocol_compliance(cls)`（`base.py:125-158`）：插件单测必调：

```python
def test_protocol_compliance():
    assert_protocol_compliance(MyProvider)
```

检查 5 个 abstract method 已实现、2 个 attr 非空、`__abstractmethods__` 已清空。失败抛 `TypeError`，破窗插件不上线。

## Registry — 进程级单例

`registry.py:23-44`：

```python
_lock = threading.Lock()
_providers: dict[str, DashboardAuthProvider] = {}

def register_provider(provider: DashboardAuthProvider) -> None:
    assert_protocol_compliance(type(provider))
    with _lock:
        if provider.name in _providers:
            raise ValueError(f"dashboard-auth provider already registered: {provider.name!r}")
        _providers[provider.name] = provider
```

- **register 时 fail-fast**：协议错误抛 `TypeError`、重名抛 `ValueError`
- **PluginContext 层做 try/except 降级为 warning**（`hermes_cli/plugins.py:582-590`）—— 破窗 plugin 不让 host 崩溃
- **list_providers / get_provider** 是 middleware 与 `/api/auth/providers` 端点的数据源

## Middleware 闸门

`middleware.py:32-46` 公开路由白名单（前缀匹配，order matters）：

```python
_GATE_PUBLIC_PREFIXES: tuple[str, ...] = (
    "/auth/login",
    "/auth/callback",
    "/auth/logout",
    "/login",
    "/api/auth/providers",
    "/assets/",
    "/favicon.ico",
    "/ds-assets/",
    "/fonts/",
    "/fonts-terminal/",
)
```

通过判定：
1. 公共路径 → 直接放行
2. 否则读 cookie → `provider.verify_session()` → ok 则 `request.state.session = Session(...)` 继续
3. 失败 + HTML 请求 → 302 → `/login?next=<原 path>`
4. 失败 + `/api/*` 请求 → 401 JSON envelope `{"error": "auth_required", "next": "..."}`

`auth_required = False` 时整个 middleware no-op，沿用 legacy `_SESSION_TOKEN` `auth_middleware`。

## Cookie 体系

`cookies.py`（234 行）：
- `session_at`：access token（短 TTL）
- `session_rt`：refresh token（长 TTL）
- `pkce_*`：PKCE verifier / state / nonce（≤10 分钟 TTL）

**`__Host-` / `__Secure-` 前缀**（`b26d81d`）：
- `__Host-session_at` —— 强制 Secure + Path=/ + 无 Domain
- `__Secure-session_at` —— 强制 Secure（可有 Domain）
- 浏览器据前缀拒绝跨子域注入，弥补反向代理 + multi-tenant 场景的 cookie 注入风险

## WebSocket 单次性 ticket

浏览器原生 WebSocket upgrade 不能带 `Authorization` 头。`ws_tickets.py` 提供独立的"鉴权后短期凭证"路径：

```python
TTL_SECONDS = 30

def mint_ticket(*, user_id: str, provider: str) -> str:
    """生成 32-byte URL-safe random ticket，单次性，30s TTL"""

def consume_ticket(ticket: str) -> Dict[str, Any]:
    """校验并 pop；过期/未知/已用都抛 TicketInvalid"""
```

- SPA 流程：`fetch POST /api/auth/ws-ticket`（带 cookie 鉴权）→ 拿 `ticket` → `wss://...?ticket=<token>` upgrade
- 错误日志只 print 前 8 字符 + `…`，secret 永不全文出现
- 4 个 WS 端点全部走 `_ws_auth_ok(request)` 校验

## Nous Portal Provider — 总是开启 + per-instance client_id

`plugins/dashboard_auth/nous/__init__.py`（582 行）实现 `nous-account-service/docs/agent-dashboard-oauth-contract.md`（PR #180）。

### 配置 surface

env > config.yaml（空值视作未设）：

```yaml
# config.yaml — canonical
dashboard:
  oauth:
    client_id: agent:{agent_instance_id}   # 必填
    portal_url: https://portal.example     # 可选

# env — 用于 Fly.io 平台 secret 注入，避免烤进 config.yaml
HERMES_DASHBOARD_OAUTH_CLIENT_ID   # 同 shape
HERMES_DASHBOARD_PORTAL_URL        # 默认 https://portal.nousresearch.com
HERMES_DASHBOARD_PUBLIC_URL        # OAuth redirect_uri 拼接的基址覆盖（公网部署）
```

### 协议要点

- **client_id per-instance**：`agent:{instance_id}` shape；token `agent_instance_id` claim 与 suffix 交叉校验 — defense-in-depth
- **scope**：仅 `agent_dashboard:access`（不带 OIDC `openid` / `profile` / `email`）
- **JWT 校验**：RS256 + `/.well-known/jwks.json` + 5 分钟 JWKS cache
- **V1 无 refresh token**：`refresh_session` 总抛 `RefreshExpiredError` → middleware 跳 `/auth/login`（contract 限制，V2 再补）

### Always-on 行为

`b3dc539`「Nous plugin always-on; default portal URL; specific error messages」：
- bundled `kind: backend` plugin auto-load
- 但仅在 `client_id` 实际配置时才 `register_provider()` — 让 loopback / `--insecure` 操作者无负担
- 验证失败时 token 的 `iss` / `aud` 透传到错误消息便于诊断（`a498485`）

## Plugin Hook 集成

`hermes_cli/plugins.py:558-594`：

```python
def register_dashboard_auth_provider(self, provider) -> None:
    """注册 dashboard 认证 provider。"""
    from hermes_cli.dashboard_auth import (
        DashboardAuthProvider, register_provider,
    )
    if not isinstance(provider, DashboardAuthProvider):
        logger.warning(
            "Plugin %r tried to register a dashboard-auth provider "
            "that does not inherit from DashboardAuthProvider. Ignoring.",
            self.manifest.name,
        )
        return
    try:
        register_provider(provider)
    except (TypeError, ValueError) as e:
        logger.warning(...)
        return
    logger.info(...)
```

错误处理统一：异常吃掉变 warning，破窗 plugin 不让 host 崩溃。

## fail-closed 与 hardening

- **无 provider 注册时拒绝启动**：dashboard 在 gated mode 下若 `list_providers()` 为空，直接 fail-closed（`53736b3`）— 防止误配置导致裸奔
- **proxy_headers 仅在 gated 时启用**：`X-Forwarded-For` / `X-Forwarded-Proto` 只在闸门启用时被信任，否则 IP spoofing 风险
- **`_SESSION_TOKEN` 不注入 SPA bundle**：gated mode 下 SPA bundle 必须依赖 cookie/ticket，避免老 token 路径误用
- **discover before serve**：`cmd_dashboard` 在 `start_server` 前 trigger plugin discovery（`4272977`）—— 否则 Nous provider 注册过晚 → fail-closed 误触发
- **`bypass loopback WS peer check` in gated mode**（`c310419`）—— gated 模式下 WS 鉴权由 ticket 提供，原 loopback 校验不适用
- **`next=` 透传**：login page + PKCE cookie 一起带 `next=<原 path>`，登录完成 redirect 回去（`034ad95`）
- **stub token 固定长度 sig 后缀**（`866cc98`）—— 防止 timing oracle 区分长度

## Phase 演进时间线（commits 同日落点）

| Phase | 描述 | 关键 commit |
|-------|------|-------------|
| 0 | `should_require_auth` + `app.state.auth_required` | `8773bbf` + `949ad95` + `f2b479e` |
| 1 | ABC + Session/LoginStart + 3 异常 | `2dc6d03` |
| 2 | Registry 注册表 + 测试 | `1bbfed7` |
| 3a | Cookie helpers + 审计日志 | `a30c4d8` + `865cae4` |
| 3b | 闸门 middleware + `/auth/*` 路由 + `/login` HTML | `5b17eab` |
| 3c | fail-closed + proxy_headers + 抑制 _SESSION_TOKEN | `53736b3` |
| 4 | Plugin Hook `register_dashboard_auth_provider` | `c32b17f` |
| 4b | Nous OAuth Provider plugin | `848baeb` |
| 5a | WS ticket 单次性 + `/api/auth/ws-ticket` | `b69fce9` |
| 5b | `_ws_auth_ok` + 4 端点接入 | `b2360ba` |
| 5c | SPA `getWsTicket()` + `buildWsAuthParam()` | `8971e94` |
| 6 | 401 re-auth envelope + `next=` 透传 | `5e9308b` |
| 7 | SPA `AuthWidget` + `/api/status` auth fields | `2fc4615` |
| 7b | `/login` 重设计为 Nous DS | `0af37ff` |
| ext | `HERMES_DASHBOARD_PUBLIC_URL` / `dashboard.public_url` | `a890389` |
| ext | canonical `dashboard.oauth` config | `61dcc33` |
| ext | `X-Forwarded-Prefix` + `__Host-` / `__Secure-` cookie | `b26d81d` |
| ext | Nous always-on + 默认 portal URL + 错误细化 | `b3dc539` |
| ext | token `iss`/`aud` 透传到错误 | `a498485` |
| ext | `cmd_dashboard` 触发 plugin discovery | `4272977` |
| docs | Phase 7 OAuth Authentication 节 + web-dashboard.md | `7c9cdbc` |

## 与 loopback `_SESSION_TOKEN` 的关系

| 模式 | auth_required | 鉴权机制 | 适用 |
|------|---|-------|------|
| loopback（默认） | False | `_SESSION_TOKEN` query/header（同源） | localhost 开发 |
| --insecure | False | 无（裸奔，仅供调试） | 本地/容器内调试 |
| 非 loopback + 未 --insecure | True | OAuth provider + cookie + WS ticket | 生产部署 / Fly.io / 反向代理后 |

老 `_SESSION_TOKEN` 路径并未被删除 — 仅在 `auth_required=False` 时启用；gated mode 下被显式抑制不注入 SPA bundle。

## 测试矩阵

`tests/plugins/dashboard_auth/` 与 `tests/test_dashboard_*` 覆盖：
- `cover registry register/get/list/clear semantics`（`1bbfed7`）
- `pin current loopback auth behavior as regression harness`（`f2b479e`）
- `stub auth provider for E2E gate testing`（`628a52f`）
- `strip HERMES_DASHBOARD_OAUTH_* env vars in hermetic fixture`（`c598076`）—— 防止 host 环境污染测试

## 2026-06-01 增量 — Dashboard 全管理面板复用既有闸门（#36704，`b571ec298`）

`hermes_cli/web_server.py +786`（6300→7079 行）新增 17 个管理端点，全部经现有 middleware 走 OAuth 闸门 —— **无新增暴露面**：

| 模块 | 端点 | 行号 |
|------|------|------|
| MCP | `GET/POST /api/mcp/servers` + `DELETE/POST /api/mcp/servers/{name}` + `.../test` | 4041 / 4053 / 4089 / 4098 |
| Pairing | `GET /api/pairing` + `approve` / `revoke` / `clear-pending` | 4147 / 4156 / 4178 / 4192 |
| Webhooks | `GET/POST /api/webhooks` + `DELETE /api/webhooks/{name}` | 4237 / 4253 / 4305 |
| Gateway | `POST /api/gateway/{start,stop,restart}` | 4328 / 4338 / 860 |
| Credentials | `GET/POST /api/credentials/pool` + `DELETE .../{provider}/{index}` | 4385 / 4412 / 4446 |
| Memory | `GET /api/memory` + `PUT /api/memory/provider` + `POST /api/memory/reset` | 4484 / 4519 / 4543 |
| Ops | `doctor` / `security-audit` / `backup` / `import` / `hooks` / `checkpoints` / `checkpoints/prune` | 4580 / 4590 / 4605 / 4622 / 4637 / 4673 / 4705 |

前端 4 个新页面 `web/src/pages/{McpPage,PairingPage,WebhooksPage,SystemPage}.tsx`（446 + 276 + 447 + 663 行）+ `web/src/App.tsx:78-81 import` + `:131-134` 路由 + `web/src/lib/api.ts +257` fetch wrappers。228 行测试 `tests/hermes_cli/test_dashboard_admin_endpoints.py`。

### Dashboard OAuth `/api/*` 不在 next= round trip（`e1eba6f8c`，#36244）

`fix(dashboard-auth): drop /api/* paths from OAuth next= round trip` —— 之前 OAuth 完成跳回 `next=` 参数，会包含 `/api/...` 路径（无 UI 重定向意义）。修复后从 next= 排除 `/api/*` 路径。

详见 [[2026-06-01-update#2-dashboard-全管理面板]]。

---

## 2026-06-02 增量 — Refresh-token cookie 轮转（#37247，`c10ccaaf`）

`feat(dashboard-auth): rotate dashboard sessions via refresh token (#37247)` —— 4 文件 +428 / -55。修复**浏览器在 `Max-Age` 后驱逐 access-token cookie**（refresh token cookie 还在）的硬 401 误判：之前 middleware 把这视为"未认证"强制完整重登；现在能用 RT 静默续期 AT。

### 缘起（commit body 实证）

老路径里 `_attempt_refresh` **只在 AT 存在但失效**时触发（即调 token-introspection 失败时）。但浏览器场景下 AT 在 Max-Age 后**直接消失**，根本没有 token 走到 introspection —— gate 看到无 cookie 就强制清掉 RT、跳 OAuth 重登。**手动 Refresh button 路径**和老回归测试都恰好让 AT 保持存在，所以这个失败模式逃过审查。

### Provider 端（`plugins/dashboard_auth/nous/__init__.py`）

| 行 | 改动 |
|---|---|
| `:244-302` | 新方法 `refresh_session(refresh_token)` — POST `grant_type=refresh_token` 到 Portal `/token` |
| `:263-266` | 空 RT **fast-fail** 不联网 `RefreshExpiredError` |
| `:269-291` | RT 在 body（Portal schema 要）**和** `x-nous-refresh-token` header（log redaction）同发 |
| `:298-302` | `400` → `RefreshExpiredError`（过期/吊销/重放检测）；网络错 → `ProviderError` |
| `:304-326` | 共享 `_token_response_to_session()` 处理器 — **参数化 exception 类型**（auth-code 用 `InvalidCodeError`，refresh 用 `RefreshExpiredError`） |
| `:207-242` | `complete_login()` 改为捕获 Portal 初次返回的 RT（**前向兼容**：缺失时存为空串） |

### Middleware 端（`hermes_cli/dashboard_auth/middleware.py`）

| 行 | 改动 |
|---|---|
| `:236` | `refreshed = _attempt_refresh(request, refresh_token=_rt)` —— **AT 缺席 + RT 存在**也走刷新路径 |
| `:255` | 成功后 `refresh_token=new_session.refresh_token` 写回新 cookie（**轮转**，旧 RT 失效） |
| `:302` | 新函数签名 `def _attempt_refresh(request, *, refresh_token)`（仅接关键字参数，避免误位置） |

**关键决策**（不变量）：
- 只有 **AT 和 RT 都缺**才硬 bounce（401，让前端跳 OAuth）
- 单独存在的 RT → 走 refresh，**省略 AT verification**（无 AT 可验）
- 死/过期 RT → `RefreshExpiredError` → fall through 到 clear-and-relogin

### 与 OAuth 一致的 `_token_response_to_session` 处理器（`:304-326`）

```python
# 形态（参数化 exception type）
def _token_response_to_session(response, expired_error_cls):
    if response.status_code == 400:
        raise expired_error_cls(...)  # InvalidCodeError 或 RefreshExpiredError
    if not response.is_success:
        raise ProviderError(...)
    return Session(
        access_token=..., refresh_token=...,  # 都可空
        ...
    )
```

—— 让 `complete_login()`（auth_code → InvalidCodeError）和 `refresh_session()`（RT → RefreshExpiredError）走同一段，**省 50+ 行重复**。

### Cookie 体系（`hermes_cli/dashboard_auth/cookies.py`）

兼容性钩子（实证）：
- `:43` 注释 — `set_session_cookies` 接受 `refresh_token=""`（contract-v1 形态，provider 没返 RT）
- `:119,129-131` 函数签名 + 行为 — 空 RT 表示 "provider 不发 RT，不要 set RT cookie"
- `:148-151` if `refresh_token`: → `cookies.set("refresh_token", refresh_token, ...)` —— 空串走 else 分支不 set

### 测试矩阵

- `tests/hermes_cli/test_dashboard_auth_401_reauth.py` **+89** — **3 个关键回归**：
  - `AT_evicted + RT_present` → 透明 refresh + 写新 cookie（200，不见 401）
  - `no_cookies` → 仍然 401 跳重登
  - `RT_only + dead_RT` → **clean 401**，不见 500（早 fail）
- `tests/plugins/dashboard_auth/test_nous_provider.py` **+113** —— RT 路径单元测试 + token-response 形态测试

### 影响范围

- **不动 OAuth 流**：初始 auth_code → token 交换路径完全没改，新 hook 只在 RT 已经存在时触发
- **不动 cookie schema**：RT cookie 已经存在了一段时间，本次只是 middleware 终于学会**用**它
- **不动 Provider ABC**：`refresh_session` 是 `base.py:119` 已声明的 abstract 方法（详见 §4），Nous Provider 终于补上 concrete 实现

详见 [[2026-06-02-update#4-dashboard-auth-refresh-token-轮转]]。

---

## 相关页面

- [[security-defense-system]] — 多层防御整体框架，dashboard OAuth 是 v0.14-late 新增的 layer
- [[hook-system-architecture]] — PluginContext 8 个 register_* 钩子全表，`register_dashboard_auth_provider:558` 是其中之一
- [[provider-plugin-system]] — 同样的 ABC + Registry + Plugin discovery 模式
- [[cli-architecture]] — `hermes dashboard` 子命令是闸门的入口
- [[2026-05-27-update]] — 落地 changelog
- [[2026-06-01-update]] — Dashboard 全管理面板（17 个新端点复用闸门）
- [[2026-06-02-update]] — Channels 页 + Admin Panel v2 + Refresh-token 轮转
