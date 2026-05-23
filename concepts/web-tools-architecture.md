---
title: Web Tools 搜索/提取架构
created: 2026-04-08
updated: 2026-05-19
type: concept
tags: [tool, toolset, architecture, component, plugin]
sources: [tools/web_tools.py, agent/web_search_provider.py, agent/web_search_registry.py, plugins/web/]
---

> **v2026.5.7 增量**：
>
> - **Per-capability backend selection**（@kshitijk4poor, #20061）—— search / extract / browse 各自独立选 backend（如 SearXNG 搜索 + Firecrawl 抽取）。
> - **新搜索 backend**：
>   - **SearXNG**（@kshitijk4poor, #20823）原生 search-only backend，配合 `searxng-search` optional skill。
>   - **Brave Search**（free tier）（commit `04193cf`）。
>   - **DDGS**（DuckDuckGo Search）（commit `04193cf`）。
> - 源码确认：`tools/web_tools.py:129` backend 列表完整为 `parallel / firecrawl / tavily / exa / searxng / brave-free / ddgs`，`tools/web_tools.py:142` `("searxng", _has_env("SEARXNG_URL"))` 注释「Free-tier backends (searxng / brave-free / ddgs) trail the paid ones so」。

# Web Tools — 搜索/提取架构

## 概述

Web Tools 提供**多后端 Web 搜索/提取/爬取**能力，所有后端对 Agent 暴露相同的 `web_search`、`web_extract`、`web_crawl` 工具接口。

自 hermes 0.14.0 起，Web 后端被迁移为**插件架构**。旧的 `tools/web_providers/` 目录（及其 `tools.web_providers.base` ABC、`tools/web_tools.py` 内的逐厂商内联实现）已**整体删除**。当前结构：

- `agent/web_search_provider.py` — `WebSearchProvider` 抽象基类（ABC），插件唯一对接面。
- `agent/web_search_registry.py` — 全局注册表与活动后端解析。
- `plugins/web/<name>/` — 各 provider 插件（内置，`kind: backend`，自动加载）。
- `tools/web_tools.py`（67KB/1551 行）— 仅保留三个工具包装器、LLM 内容处理引擎、安全层；不再含任何厂商内联代码。

核心理念：**内容获取优先于浏览器自动化**——简单信息检索使用 web_search/web_extract（更快、更便宜），仅在需要交互时才使用 browser 工具。

## 插件架构

### WebSearchProvider ABC

定义于 `agent/web_search_provider.py:63`。子类必须实现 `name`（`agent/web_search_provider.py:75`）与 `is_available()`（`:90`），并实现 `search` / `extract` / `crawl` 中至少一个。能力标志位让注册表为每次调用路由到正确的 provider：

| 方法 | 默认值 | 说明 |
|---|---|---|
| `supports_search()` | `True` | 实现 `search()` 时返回 True |
| `supports_extract()` | `False` | 实现 `extract()` 时返回 True |
| `supports_crawl()` | `False` | 实现 `crawl()` 时返回 True |

```python
def search(self, query: str, limit: int = 5) -> Dict[str, Any]: ...
def extract(self, urls: List[str], **kwargs: Any) -> Any: ...
def crawl(self, url: str, **kwargs: Any) -> Any: ...
```

**async-extract 语义**：`extract()` 与 `crawl()` 均**可声明为 `async def`**。分发器通过 `inspect.iscoroutinefunction()` 检测协程并按需 `await`（`tools/web_tools.py:975`、`:1259`）。`is_available()` 必须是廉价检查（env 变量/可选依赖/实例 URL），**不得发起网络调用**——它在工具注册时及每次 `hermes tools` 刷新时运行。

### register_web_search_provider() 门面

插件通过 `PluginContext.register_web_search_provider()`（`hermes_cli/plugins.py:585`）注册 provider 实例；该门面转发到 `agent.web_search_registry.register_provider()`。**插件是 Web provider 的唯一来源**——旧的硬编码 picker 行与 skiplist 已移除。

### 七大 Provider 插件

`plugins/web/` 下现有 7 个内置 provider 插件（`plugin.yaml` 中 `kind: backend`）：

| Provider | name | Search | Extract | Crawl | 认证 / 依赖 |
|---|---|---|---|---|---|
| **Firecrawl** | `firecrawl` | ✅ | ✅ (async) | ✅ (async) | API Key 或 Nous Tool Gateway |
| **Tavily** | `tavily` | ✅ | ✅ | ✅ | `TAVILY_API_KEY` |
| **Exa** | `exa` | ✅ | ✅ | ❌ | `EXA_API_KEY` |
| **Parallel** | `parallel` | ✅ | ✅ (async) | ❌ | `PARALLEL_API_KEY` |
| **SearXNG** | `searxng` | ✅ | ❌ | ❌ | `SEARXNG_URL`（自托管实例） |
| **Brave (Free)** | `brave-free` | ✅ | ❌ | ❌ | `BRAVE_SEARCH_API_KEY`（免费层） |
| **DDGS** | `ddgs` | ✅ | ❌ | ❌ | 无 Key，需 `ddgs` Python 包 |

**Crawl 仅 Firecrawl 与 Tavily 支持**（`agent/web_search_registry.py:251`）。注意差异：Tavily 的 crawl 支持自然语言 `instructions`；Firecrawl 的 `/crawl` 端点不接受 instructions（属 `/extract` 功能），其插件会记录并丢弃该参数（`plugins/web/firecrawl/provider.py:610`）。

### 注册表分发路径

`tools/web_tools.py` 的三个工具包装器不再含厂商逻辑，仅做注册表查找 + 委派：

```
web_search_tool   → _get_search_backend() → web_search_registry.get_provider()
                    → 回退 get_active_search_provider()  → provider.search()
web_extract_tool  → get_active_extract_provider()        → provider.extract()  [可 await]
web_crawl_tool    → get_active_crawl_provider()          → provider.crawl()    [可 await]
```

### 活动后端解析

`_resolve()`（`agent/web_search_registry.py:133`）按以下优先级选定活动 provider：

1. 显式配置 `web.<capability>_backend`（`search_backend` / `extract_backend` / `crawl_backend`）。
2. 共享回退 `web.backend`。
3. **单 provider 捷径**：当某能力恰有一个已注册且 `is_available()` 的 provider 时直接选用。
4. **遗留偏好顺序**（按可用性过滤）：`firecrawl → parallel → tavily → exa → searxng → brave-free → ddgs`，与迁移前 `_get_backend()` 顺序一致——付费后端在前，确保升级后既有付费配置不会被降级到免费层。
5. 否则返回 `None`，工具向用户报错并指引 `hermes tools`。

每一步都施加能力过滤（`supports_search/extract/crawl`），因此把仅支持搜索的 provider（如 `brave-free`）配为 `web.extract_backend` 会正确穿透到具备 extract 能力的后端。注意：**显式配置即使 `is_available()` 为 False 也会被选中**，以便分发器给出精确的「X_API_KEY 未设置」错误而非静默切换。

### Firecrawl 双路径认证

Firecrawl 是默认后端，支持两种连接模式（`plugins/web/firecrawl/provider.py:205` `_get_firecrawl_client`）：

| 模式 | 配置 | 适用对象 |
|---|---|---|
| **直接模式** | `FIRECRAWL_API_KEY` / `FIRECRAWL_API_URL`（自托管） | 所有用户 |
| **托管 Gateway** | Nous Tool Gateway（`FIRECRAWL_GATEWAY_URL` / `TOOL_GATEWAY_*`） | Nous 订阅者 |

当两者均配置时，`web.use_gateway: true` 会优先走托管 Gateway，否则直接模式优先（`provider.py:226`）。`check_firecrawl_api_key()` 在任一路径可用时返回 True。客户端实例被缓存在 `tools.web_tools` 模块上（`_firecrawl_client` / `_firecrawl_client_config`），配置不变时复用。

**优越性**：Nous 订阅者无需单独购买 Firecrawl，通过 tool-gateway 共享访问。

## 核心工具

### web_search_tool — 网络搜索

```python
def web_search_tool(query: str, limit: int = 5) -> str:
```

`limit` 被钳制到 1–100。返回统一格式：
`{"success": true, "data": {"web": [{"title", "url", "description", "position"}]}}`。
所有 provider 的 `search()` 均为同步实现。

### web_extract_tool — URL 内容提取

```python
async def web_extract_tool(
    urls: List[str],
    format: str = None,            # markdown 或 html，None 时由 provider 决定
    use_llm_processing: bool = True,
    model: Optional[str] = None,
    min_length: int = 5000         # 触发 LLM 处理的最小长度
) -> str:
```

**核心流程**：
1. 安全检查（密钥注入 + SSRF `is_safe_url()` + 网站策略）
2. 后端提取——注册表选定的 provider 的 `extract()`（同步或 `await`）
3. LLM 智能压缩（`process_content_with_llm`）
4. 输出裁剪 + `clean_base64_images()`

### web_crawl_tool — 网站爬取

```python
async def web_crawl_tool(
    url: str,
    instructions: str = None,    # 自然语言提取指令（仅 Tavily 支持）
    depth: str = "basic",        # basic 或 advanced
    use_llm_processing: bool = True,
    model: Optional[str] = None,
    min_length: int = 5000
) -> str:
```

分发器先对种子 URL 做 SSRF + 网站策略校验，再委派给 `get_active_crawl_provider()`。当无 crawl-capable provider 时，分发器回退到辅助模型摘要路径。

## LLM 内容处理引擎

这是 Web Tools 最具创新性的部分——用 LLM 自动压缩网页内容（`process_content_with_llm`，`tools/web_tools.py:320`）。

### 处理策略

源码常量（`tools/web_tools.py:348`）：

```python
MAX_CONTENT_SIZE = 2_000_000   # 2M chars — 超过则完全拒绝
CHUNK_THRESHOLD  = 500_000     # 500k chars — 超过则分块处理
CHUNK_SIZE       = 100_000     # 每块 100k chars
MAX_OUTPUT_SIZE  = 5000        # 最终输出硬上限
DEFAULT_MIN_LENGTH_FOR_SUMMARIZATION = 5000  # 低于此值跳过处理
```

- `< 5000 chars` → 跳过处理，直接返回原始内容
- `5000 ~ 500K chars` → 单次 LLM 摘要
- `500K ~ 2M chars` → 分块处理 + 合成
- `> 2M chars` → 拒绝处理

### 分块处理（Chunked Processing）

`_process_large_content_chunked`（`tools/web_tools.py:538`）：

1. 将内容切分为 100K chars 的块
2. 并行摘要每个块（`asyncio.gather`）
3. 合成所有块摘要为统一摘要
4. 硬性限制：最终输出 ≤ 5000 chars

**设计亮点**：
- 每个块使用**专门的 prompt**（"这是大文档的一节，不要写引言和结论"）
- 并行处理所有块，不串行等待
- 合成步骤**去除冗余**并整合为连贯摘要
- 如果合成失败，**回退为拼接所有块摘要**

## 安全设计

| 层级 | 保护 | 实现 |
|---|---|---|
| **URL 密钥注入** | 阻止 URL 中嵌入 API Key | 提取前检查嵌入密钥 |
| **SSRF 防护** | 阻止访问私有地址 | `is_safe_url()`（`tools/url_safety.py`） |
| **网站策略** | 黑名单域名拦截 | `check_website_access()`（`tools/website_policy.py`） |
| **重定向检查** | 阻止重定向到内部地址 | provider 提取后按 `sourceURL` 重新校验策略 |

策略校验下沉到各 provider 插件：Firecrawl 插件在抓取前后、爬取每页时都重新调用 `check_website_access()`，命中时返回带 `blocked_by_policy` 字段的结果项而非抛异常。

### Base64 图片清理

```python
def clean_base64_images(text: str) -> str:
    """移除 base64 编码图片，替换为占位符"""
    # 防止大量 base64 数据挤占上下文窗口
```

## Debug 模式

```bash
export WEB_TOOLS_DEBUG=true
```

启用后通过 `DebugSession`（`tools/web_tools.py:317`）记录所有工具调用、参数、响应大小、LLM 压缩指标与最终结果。

## 配置与操作

### 选择后端

```yaml
# config.yaml
web:
  backend: firecrawl         # 共享回退：所有能力
  search_backend: searxng    # 可选：仅 web_search 用 SearXNG
  extract_backend: firecrawl # 可选：仅 web_extract 用 Firecrawl
  crawl_backend: tavily      # 可选：仅 web_crawl 用 Tavily
  use_gateway: false         # Firecrawl 双路径时优先托管 Gateway
```

推荐通过 `hermes tools` 交互式配置（picker 行由各插件的 `get_setup_schema()` 动态生成）。

### 环境变量

```bash
# Firecrawl 直接模式
export FIRECRAWL_API_KEY=fc-xxx
export FIRECRAWL_API_URL=https://your-self-hosted.com   # 自托管，可选
# Tavily / Exa / Parallel
export TAVILY_API_KEY=tav-xxx
export EXA_API_KEY=exa-xxx
export PARALLEL_API_KEY=par-xxx
# SearXNG / Brave (Free)
export SEARXNG_URL=https://your-searxng-instance
export BRAVE_SEARCH_API_KEY=brave-xxx
# LLM 处理配置
export AUXILIARY_WEB_EXTRACT_MODEL=google/gemini-3-flash-preview
```

### 禁用 LLM 处理

```python
# 快速提取，不需要压缩
content = await web_extract_tool(["https://example.com"], use_llm_processing=False)
```

## 设计优越性

- **可插拔后端**：新增 provider 只需在 `plugins/web/` 下放一个实现 `WebSearchProvider` ABC 的插件，无需改动 `web_tools.py`。
- **按能力分路由**：search / extract / crawl 可分别配置不同后端。
- **内容压缩**：自动 LLM 摘要 + 大内容分块合成，节省上下文窗口。
- **安全防护**：SSRF + 密钥注入 + 网站策略 + 重定向重检。
- **格式统一**：所有 provider 遵守同一响应契约，Agent 永远收到统一格式。

## 与其他系统的关系

- [[auxiliary-client-architecture]] — LLM 内容处理通过辅助客户端调用摘要模型
- [[tool-registry-architecture]] — web_search/web_extract/web_crawl 通过 registry 注册
- [[browser-tool-architecture]] — 文档建议简单信息获取优先用 web_tools
- [[context-compressor-architecture]] — 类似的 LLM 压缩理念应用于不同场景
