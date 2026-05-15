---
title: Web Tools 搜索/提取架构
created: 2026-04-08
updated: 2026-05-15
type: concept
tags: [tool, toolset, architecture, component, plugin]
sources: [tools/web_tools.py, agent/web_search_provider.py, agent/web_search_registry.py, plugins/web/]
---

# Web Tools — 搜索/提取架构

## 概述

Web Tools 提供**多后端 Web 搜索/提取/爬取**能力，对 Agent 暴露统一的 `web_search`、`web_extract`、`web_crawl` 工具接口。

> **架构变更（2026-05-13 重构）**：所有 Web 后端已迁移到**插件架构**。旧的 in-tree 目录 `tools/web_providers/` 已被删除，每个后端现在是 `plugins/web/<name>/` 下的独立插件，实现统一的 `WebSearchProvider` ABC。`tools/web_tools.py`（约 67KB）现在只负责安全检查、LLM 内容处理与**注册表分发**，不再包含任何 per-vendor 内联代码。

核心理念：**内容获取优先于浏览器自动化**——简单信息检索使用 web_search/web_extract（更快、更便宜），仅在需要交互时才使用 browser 工具。

## 架构原理

### 插件化的七大后端

所有后端均为 `plugins/web/<name>/` 下的内置插件（`kind: backend`，自动加载），每个插件包含 `plugin.yaml`、`provider.py` 与 `__init__.py`。在 `plugin.yaml` 中通过 `provides_web_providers` 声明。

| 后端 (`name`) | Search | Extract | Crawl | 认证 / 依赖 |
|---|---|---|---|---|
| **firecrawl** | ✅ | ✅ | ✅ | `FIRECRAWL_API_KEY` / `FIRECRAWL_API_URL` 或 Nous Gateway |
| **parallel** | ✅ | ✅ | ❌ | `PARALLEL_API_KEY` |
| **tavily** | ✅ | ✅ | ✅ | `TAVILY_API_KEY` |
| **exa** | ✅ | ✅ | ❌ | `EXA_API_KEY` |
| **searxng** | ✅ | ❌ | ❌ | `SEARXNG_URL`（自托管） |
| **brave-free** | ✅ | ❌ | ❌ | `BRAVE_SEARCH_API_KEY`（免费层） |
| **ddgs** | ✅ | ❌ | ❌ | 无需 Key，`pip install ddgs` |

仅 **firecrawl** 与 **tavily** 原生支持 crawl。`searxng`、`brave-free`、`ddgs` 为 search-only 后端。

### WebSearchProvider ABC

定义于 `agent/web_search_provider.py`，是 Web 后端**唯一的插件对接面**（镜像 image_gen 模板）。子类必须实现 `name` 与 `is_available()`，并至少实现 `search` / `extract` / `crawl` 之一。

```python
class WebSearchProvider(abc.ABC):
    @property
    @abc.abstractmethod
    def name(self) -> str: ...          # web.*_backend 配置键匹配的稳定标识
    @abc.abstractmethod
    def is_available(self) -> bool: ... # 廉价检查，禁止网络调用
    def supports_search(self) -> bool: return True
    def supports_extract(self) -> bool: return False
    def supports_crawl(self) -> bool: return False
    def search(self, query, limit=5) -> Dict: ...
    def extract(self, urls, **kwargs) -> Any: ...   # 可为 async def
    def crawl(self, url, **kwargs) -> Any: ...      # 可为 async def
    def get_setup_schema(self) -> Dict: ...         # hermes tools picker 元数据
```

`supports_*` 能力标志让注册表把每个工具调用路由到正确的后端；多能力后端（Firecrawl/Tavily/Exa…）可从单一类同时声明多个能力。`extract` 与 `crawl` 既可为同步也可为 `async def`——分发器通过 `inspect.iscoroutinefunction` 检测并 `await`，同步实现则用 `asyncio.to_thread` 包裹。

### 注册表与插件 facade

- **注册表** `agent/web_search_registry.py`：维护 `name → provider` 的全局映射（线程安全），插件在 import 期注册。
- **插件 facade** `PluginContext.register_web_search_provider()`（定义于 `hermes_cli/plugins.py`）：插件用它注册 provider，会校验实例继承自 `WebSearchProvider`，再调用注册表的 `register_provider()`。

### 后端选择链（活跃 provider 解析）

注册表通过 `get_active_search_provider()` / `get_active_extract_provider()` / `get_active_crawl_provider()` 按能力解析活跃后端，优先级如下：

```text
1. web.{capability}_backend  (per-capability 覆盖: search_backend/extract_backend/crawl_backend)
2. web.backend               (共享 fallback)
3. 若仅有一个支持该能力且 is_available() 的 provider → 直接选用
4. Legacy 偏好顺序 (按 is_available 过滤):
   firecrawl → parallel → tavily → exa → searxng → brave-free → ddgs
5. 否则返回 None — 工具提示运行 `hermes tools` 配置后端
```

显式 config 命中时**忽略可用性**直接返回，使分发器能给出精确的 "X_API_KEY 未设置" 错误，而不是静默切换后端。能力过滤在每一步生效——例如把 search-only 的 `brave-free` 配成 `web.extract_backend` 时会正确回退到支持 extract 的后端。`tools/web_tools.py` 内的 `_get_backend()` / `_get_capability_backend()` 保留同样的 legacy 候选顺序以保持向后兼容。

### Firecrawl 双路径架构

Firecrawl 是 legacy 偏好顺序中的最高优先后端，支持两种连接模式（实现位于 `plugins/web/firecrawl/provider.py`）：

| 模式 | 路径 / 环境变量 | 适用对象 |
|---|---|---|
| **直接模式** | `FIRECRAWL_API_KEY` / `FIRECRAWL_API_URL`（自托管） | 所有用户 |
| **托管 Gateway** | `FIRECRAWL_GATEWAY_URL` + Nous 订阅者 token | Nous 订阅者 |

`_get_firecrawl_client()` 在两种凭证同时存在时由 `web.use_gateway` 配置决定优先级；客户端按配置缓存，配置不变时复用同一实例。`is_available()` 在直接配置或 gateway 就绪任一满足时返回 True。

**优越性**：Nous 订阅者无需单独购买 Firecrawl，通过 tool-gateway 共享访问。

## 核心组件

`tools/web_tools.py` 中的三个工具入口现在都是**注册表查找 + 委派**——具体后端逻辑全部在 `plugins/web/<name>/provider.py` 中。

### 1. web_search_tool — 网络搜索

```python
def web_search_tool(query: str, limit: int = 5) -> str:
    # 通过 agent.web_search_registry 分发:
    #   backend = _get_search_backend()
    #   provider = get_provider(backend) 或 fallback get_active_search_provider()
    #   response = provider.search(query, limit)
```

所有 7 个 provider 的 `search()` 均为同步实现。返回统一格式：`{"success": true, "data": {"web": [{"title", "url", "description", "position"}]}}`。无 provider 时返回错误，提示运行 `hermes tools`。

### 2. web_extract_tool — URL 内容提取

```python
async def web_extract_tool(
    urls: List[str],
    format: str = "markdown",      # markdown 或 html
    use_llm_processing: bool = True,
    model: Optional[str] = None,
    min_length: int = 5000         # 触发 LLM 处理的最小长度
) -> str:
```

**核心流程**：
1. 安全检查（密钥注入 + SSRF + 网站策略）
2. 注册表解析活跃 extract provider，调用 `provider.extract(urls, **kwargs)`——分发器用 `inspect.iscoroutinefunction` 检测并按需 `await`（Parallel 的 SDK 原生异步，Exa/Tavily/Firecrawl 同步）
3. LLM 智能压缩（`process_content_with_llm`）
4. 输出裁剪（只保留 url/title/content/error）

provider 的 `extract()` 返回统一的 `[{"url", "title", "content", "raw_content", "metadata"?, "error"?}, ...]` 列表。

### 3. web_crawl_tool — 网站爬取

```python
async def web_crawl_tool(
    url: str,
    instructions: str = None,    # 提取指令（仅 Tavily 支持）
    depth: str = "basic",        # basic 或 advanced
    use_llm_processing: bool = True
) -> str:
```

仅 **firecrawl** 和 **tavily** 的 provider 声明 `supports_crawl()=True`。分发逻辑：
- 配置的后端支持 crawl → 通过注册表 `get_active_crawl_provider()` 调用 `provider.crawl()`（同步/异步均可）
- 配置的后端 search-only（brave-free/ddgs/searxng，连 extract 也不支持）→ 返回 typed "search-only" 错误
- provider 被配置但不可用（如缺 `FIRECRAWL_API_KEY`）→ 提前短路返回顶层错误信封

crawl provider 返回 `{"results": [{"url", "title", "content", ...}]}`，再交由共享的 LLM 摘要后处理。

## LLM 内容处理引擎

这是 Web Tools 最具创新性的部分——用 LLM 自动压缩网页内容。

### 处理策略

```python
def process_content_with_llm(content, url, title, model, min_length):
    """
    内容分级处理:
    < 5000 chars → 跳过处理，直接返回原始内容
    5000 ~ 500K chars → 单次 LLM 摘要
    500K ~ 2M chars → 分块处理 + 合成
    > 2M chars → 拒绝处理
    """
```

### 分块处理（Chunked Processing）

```python
async def _process_large_content_chunked(content, chunk_size=100K):
    # 1. 将内容切分为 100K chars 的块
    # 2. 并行摘要每个块 (asyncio.gather)
    # 3. 合成所有块摘要为统一摘要
    # 4. 硬性限制: 最终输出 ≤ 5000 chars
```

**设计亮点**：
- 每个块使用**专门的 prompt**（"这是大文档的一节，不要写引言和结论"）
- 并行处理所有块，不串行等待
- 合成步骤**去除冗余**并整合为连贯摘要
- 如果合成失败，**回退为拼接所有块摘要**

### 压缩率

典型压缩比：10-50x（原始内容 → LLM 摘要）

```
原始: 50,000 chars → 处理后: 2,000 chars (4%)
原始: 200,000 chars → 处理后: 4,500 chars (2.25%)
```

## 安全设计

### 四层防护

| 层级 | 保护 | 实现 |
|---|---|---|
| **URL 密钥注入** | 阻止 URL 中嵌入 API Key | `_PREFIX_RE` 检测 |
| **SSRF 防护** | 阻止访问私有地址 | `is_safe_url()` |
| **网站策略** | 黑名单域名拦截 | `check_website_access()` |
| **重定向检查** | 阻止重定向到内部地址 | 提取后检查 `sourceURL` |

### Base64 图片清理

```python
def clean_base64_images(text: str) -> str:
    """移除 base64 编码图片，替换为 [BASE64_IMAGE_REMOVED]"""
    # 防止大量 base64 数据挤占上下文窗口
```

## 标准化层

不同后端返回不同的数据格式。每个 provider 在其插件模块内部完成标准化，使所有后端对外暴露 ABC 中定义的统一响应形状。重构后这些 vendor helper 已从 `web_tools.py` 移入各插件，再由 `web_tools.py` 重新 import 以保持兼容：

```python
# plugins/web/firecrawl/provider.py
_extract_web_search_results(response)    # Firecrawl 多格式提取
_to_plain_object(value)                  # SDK 对象 → Python dict
_normalize_result_list(values)           # 混合 SDK/list → dict list

# plugins/web/tavily/provider.py
_normalize_tavily_search_results(raw)    # Tavily → 标准格式
_normalize_tavily_documents(raw)         # Tavily extract/crawl → 标准格式
```

`WebSearchProvider` ABC 把响应形状契约写进文档并逐位保留旧的 in-tree 契约，因此工具 wrapper 无需做任何翻译。

**优越性**：Agent 永远收到统一格式的数据，不需要根据后端类型做不同的解析。

## Debug 模式

```bash
export WEB_TOOLS_DEBUG=true
```

启用后自动记录：
- 所有工具调用及参数
- 原始 API 响应
- LLM 压缩指标（原始大小/处理后大小/压缩比）
- 最终处理结果

日志保存到：`~/.hermes/logs/web_tools_debug_UUID.json`

## 设计优越性

### 对比直接调用 API

| 维度 | 直接调用 API | Web Tools |
|---|---|---|
| 后端切换 | 需要改代码 | config.yaml 一键切换 |
| 内容压缩 | 手动处理 | 自动 LLM 摘要 |
| 大内容处理 | 容易超上下文 | 分块 + 合成 |
| 安全防护 | 需要自己实现 | SSRF + 注入 + 策略三层防护 |
| 格式统一 | 每个 API 格式不同 | 统一输出格式 |
| 调试 | 需要手动打印 | 内置 Debug 模式 |

### LLM 处理的优越性

没有 LLM 处理时，Agent 收到的是原始 HTML/markdown 全文（可能数十万字）。有了 LLM 处理后：
- **上下文节省**：压缩 10-50x
- **信息密度提升**：只保留关键事实和数据
- **格式统一**：所有页面都是结构化 Markdown 摘要
- **优雅降级**：LLM 失败时回退为截断原始内容

## 配置与操作

### 选择后端

```yaml
# config.yaml
web:
  backend: firecrawl          # 共享 fallback (firecrawl/parallel/tavily/exa/searxng/brave-free/ddgs)
  search_backend: searxng     # 可选: 仅 web_search 用此后端
  extract_backend: firecrawl  # 可选: 仅 web_extract 用此后端
  crawl_backend: tavily       # 可选: 仅 web_crawl 用此后端
```

per-capability 键允许搜索与提取使用不同后端（例如 SearXNG 搜索 + Firecrawl 提取）。

### 环境变量

```bash
# Firecrawl 直接模式
export FIRECRAWL_API_KEY=fc-xxx
export FIRECRAWL_API_URL=https://your-self-hosted.com  # 可选

# Exa
export EXA_API_KEY=exa-xxx

# Parallel
export PARALLEL_API_KEY=par-xxx

# Tavily
export TAVILY_API_KEY=tav-xxx

# SearXNG (自托管 metasearch)
export SEARXNG_URL=https://your-searxng-instance

# Brave Search (免费层, 2k 查询/月)
export BRAVE_SEARCH_API_KEY=brave-xxx

# ddgs (DuckDuckGo) — 无需 Key, 仅需安装包

# LLM 处理配置
export AUXILIARY_WEB_EXTRACT_MODEL=google/gemini-3-flash-preview
```

### 禁用 LLM 处理

```python
# 快速提取，不需要压缩
content = await web_extract_tool(["https://example.com"], use_llm_processing=False)
```

## 与其他系统的关系

- [[mcp-and-plugins]] — 七大 Web 后端均为 `kind: backend` 内置插件，通过 `ctx.register_web_search_provider()` facade 注册
- [[auxiliary-client-architecture]] — LLM 内容处理通过 `async_call_llm(task="web_extract")` 调用
- [[tool-registry-architecture]] — web_search/web_extract 通过 registry 注册
- [[browser-tool-architecture]] — 文档建议简单信息获取优先用 web_tools
- [[context-compressor-architecture]] — 类似的 LLM 压缩理念应用于不同场景
