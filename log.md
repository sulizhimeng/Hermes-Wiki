# Wiki Log

> 所有 wiki 操作的按时间顺序记录。只追加，不修改。
> 格式：`## [YYYY-MM-DD] action | subject`
> Actions: ingest, update, query, lint, create, archive, delete
> 当此文件超过 500 条时，轮换：重命名为 log-YYYY.md，重新开始。

## [2026-04-07] create | Wiki initialized
- Domain: Hermes Agent — Skills System and Memory
- Structure created with SCHEMA.md, index.md, log.md
- Directory structure: raw/{articles,papers,transcripts,assets}, entities, concepts, comparisons, queries

## [2026-04-07] create | 批量创建 8 个 wiki 页面
基于对 hermes-agent 代码库的深入分析（814 个文件）：

**概念页 (6):**
- skills-system-architecture — 渐进式披露架构
- memory-system-architecture — 冻结快照模式
- agent-loop-and-prompt-assembly — Agent 循环和提示构建
- skills-and-memory-interaction — Skills 与 Memory 的互补关系
- toolsets-system — 工具分组系统
- session-search-and-sessiondb — FTS5 跨会话搜索
- messaging-gateway-architecture — 14+ 平台统一网关
- context-compression — 上下文压缩

**实体页 (2):**
- aiagent-class — AIAgent 核心类
- memorystore-class — MemoryStore 核心类

## [2026-04-08] create | 16 个专题系统分析
基于源码深入分析，完成 16 个专题：

**核心架构 (6):**
- skills-system-architecture — 渐进式披露架构
- memory-system-architecture — 冻结快照模式
- agent-loop-and-prompt-assembly — Agent 循环和提示构建
- skills-and-memory-interaction — Skills 与 Memory 的互补关系
- toolsets-system — 工具分组系统
- session-search-and-sessiondb — FTS5 跨会话搜索

**性能与优化 (5):**
- parallel-tool-execution — 智能并发安全检测
- prompt-caching-optimization — Anthropic 缓存策略
- fuzzy-matching-engine — 8 策略链模糊匹配
- model-metadata-and-routing — 模型元数据缓存
- large-tool-result-handling — 大型结果处理

**安全与可靠性 (4):**
- security-defense-system — 5 层防御体系
- interrupt-and-fault-tolerance — 中断传播与容错
- credential-pool-and-isolation — 凭证池与隔离
- iteration-budget-and-delegation — 迭代预算与委派

**平台与扩展 (7):**
- cli-architecture — CLI 架构
- gateway-multi-platform — 多平台网关
- configuration-and-profiles — 配置与 Profile
- mcp-and-plugins — MCP 与插件
- terminal-backends — 终端后端
- cron-scheduling — Cron 调度
- trajectory-and-data-generation — 轨迹保存

- index.md 更新为 24 页，按类别组织
- SCHEMA.md 已定义完整标签分类法

## [2026-04-08] update | Tool Registry 工具注册系统 wiki 页面创建
- 文件: concepts/tool-registry-architecture.md
- 源码: tools/registry.py (10KB/275行)
- 核心内容: 中央工具注册系统，声明式注册+集中调度，循环导入安全设计，__slots__ 内存优化，MCP 动态注销支持
- 创建 cron job: Hermes Wiki 专题编写（每小时一个），8 次重复
- index.md 更新为 25 页

## [2026-04-08] update | Auxiliary Client 辅助客户端 wiki 页面创建
- 文件: concepts/auxiliary-client-architecture.md
- 源码: agent/auxiliary_client.py (85KB/2127行)
- 核心内容: 辅助 LLM 客户端路由器，多 provider 解析链（8 级降级）、适配器模式（Codex/Anthropic 统一为 chat.completions 接口）、客户端缓存+事件循环安全、支付/配额耗尽自动降级、任务级独立配置
- index.md 更新为 26 页

## [2026-04-08] update | Browser Tool 浏览器自动化 wiki 页面创建
- 文件: concepts/browser-tool-architecture.md
- 源码: tools/browser_tool.py (84KB/2202行)
- 核心内容: 多后端浏览器自动化（本地/Cloud/CDP/Camofox），accessibility tree 文本化页面表示，三层安全防护（SSRF/注入/策略），并发会话隔离（独立 socket 目录），后台清理线程+atexit 双重保障
- index.md 更新为 27 页

## [2026-04-08] update | Web Tools 搜索/提取 wiki 页面创建
- 文件: concepts/web-tools-architecture.md
- 源码: tools/web_tools.py (85KB/2099行)
- 核心内容: 多后端搜索/提取/爬取（Firecrawl/Exa/Parallel/Tavily），LLM 智能内容压缩（单次+分块并行+合成），Firecrawl 双路径架构（直接 API + Nous Gateway），四层安全防护，标准化层统一输出格式
- index.md 更新为 28 页

## [2026-04-08] update | 双专题 wiki 页面创建（Prompt Builder + Context Compressor）
- 文件: concepts/prompt-builder-architecture.md
  - 源码: agent/prompt_builder.py (40KB/959行)
  - 核心内容: 系统提示模块化组装，上下文文件注入防护（10种威胁模式+11种不可见Unicode），技能索引缓存+快照持久化，平台提示适配，模型特定执行指导
- 文件: concepts/context-compressor-architecture.md
  - 源码: agent/context_compressor.py (30KB/696行)
  - 核心内容: 自动上下文压缩，结构化摘要模板（Goal/Progress/Decisions/Files/Next Steps），迭代更新，工具输出修剪，工具调用对完整性保障，失败冷却
- index.md 更新为 30 页

## [2026-04-08] update | 最后 2 专题 wiki 页面创建（Model Tools + Gateway Session）
- 文件: concepts/model-tools-dispatch.md
  - 源码: model_tools.py (22KB/577行，从 2400 行重构而来)
  - 核心内容: 工具编排与调度，异步桥接三种路径，动态 schema 调整（execute_code/browser_navigate），参数类型强制，Agent 级工具拦截，三层工具发现机制
- 文件: concepts/gateway-session-management.md
  - 源码: gateway/session.py (41KB/1081行)
  - 核心内容: 多平台会话管理，SessionSource 统一抽象，SessionKey 构建规则，PII 脱敏（Discord 例外），动态系统提示注入，双存储策略（SQLite+JSON），原子保存，会话重置策略
- index.md 更新为 32 页

## [2026-04-08] update | 多 Agent 体系 wiki 页面创建
- 文件: concepts/multi-agent-architecture.md
- 源码: tools/delegate_tool.py (40KB/978行), batch_runner.py (54KB/1285行), tools/send_message_tool.py (39KB/952行)
- 核心内容: 分层子代理委派（单任务/并行最多3个），安全沙箱（5类禁止工具+深度限制），凭证继承与池共享，ACP 异构 Agent 编排，批量处理引擎，跨平台消息投递
- index.md 更新为 35 页
## [2026-04-08] update | 三专题 wiki 页面创建（Prompt Caching + Smart Model Routing + Hook System）
- 文件: concepts/prompt-caching-optimization.md (updated)
  - 源码: agent/prompt_caching.py (2KB/72行)
  - 核心内容: Anthropic system_and_3 缓存策略，4断点滚动窗口，纯函数设计
- 文件: concepts/smart-model-routing.md
  - 源码: agent/model_metadata.py (36KB/941行), agent/models_dev.py (25KB/781行), hermes_cli/model_switch.py (32KB/927行)
  - 核心内容: 10级上下文长度解析链，models.dev 4000+模型数据库，本地服务器自动探测(4种)，别名系统，token估算
- 文件: concepts/hook-system-architecture.md
  - 源码: gateway/hooks.py (170行), hermes_cli/plugins.py (609行)
  - 核心内容: Gateway Hooks 事件驱动(8种事件+通配符)，Plugin System 三级来源(用户/项目/pip)，PluginContext API(工具注册/消息注入/CLI命令/钩子)，缓存友好上下文注入
- index.md 更新为 37 页

## [2026-05-20] ingest+update | 同步 v0.12.0/v0.13.0/v0.14.0 三大版本（~2480 commits）

Wiki 之前停在 2026-04-29，对应 v0.11.0 (v2026.4.23)。本期同步到 hermes-agent HEAD = `31a010010 feat(state.db): persist platform_message_id; restore yuanbao exact-id recall` (2026-05-20)，跨过三个发布：

- v0.12.0 (v2026.4.30) — Curator Release，1096 commits
- v0.13.0 (v2026.5.7)  — Tenacity Release（Kanban / `/goal` / 安全潮 8 P0），864 commits
- v0.14.0 (v2026.5.16) — Foundation Release（PyPI / 减肥 / 冷启动 −19s / Grok 1M），808 commits

**新增 6 个概念页**（源码全部 grep 确认）：

- concepts/multi-agent-kanban.md
  - 源码: hermes_cli/kanban*.py (~11200 行) + tools/kanban_tools.py (1297 行)
  - 核心内容: SQLite 板 6 张表 / 9 个 worker tool (HERMES_KANBAN_TASK + orchestrator 双门控) / 自动分解 / Swarm 拓扑 / 1058 行诊断规则引擎
- concepts/goal-loop-and-steering.md
  - 源码: hermes_cli/goals.py (762 行)
  - 核心内容: Ralph 循环 / continuation = 普通 user message / judge fail-OPEN / 用户消息抢占 / `/steer` next-turn note / `/queue` FIFO / `/handoff` 现场迁移
- concepts/provider-plugin-system.md
  - 源码: providers/base.py:39 ProviderProfile / providers/__init__.py / plugins/model-providers/ × 29
  - 核心内容: 纯数据 dataclass + 5 个钩子 / 用户 last-writer-wins / 与 Transport 的 api_mode 耦合点 / pkgutil 兼容
- concepts/hermes-proxy.md
  - 源码: hermes_cli/proxy/{server.py:308, cli.py:142, adapters/{base,nous_portal,xai}.py:436}
  - 核心内容: 凭证-attach 转发器 / 仅替换 Authorization / 路径白名单 / 每请求 resolve() OAuth refresh / SSE 原样转发
- concepts/lsp-integration.md
  - 源码: agent/lsp/* (11 文件) + tools/file_operations.py
  - 核心内容: git-workspace 门控 / delta-only 诊断 / range_shift / per-workspace daemon / 失败回退原 in-process 语法检查
- concepts/i18n-and-locales.md
  - 源码: agent/i18n.py + locales/*.yaml (16 个)
  - 核心内容: 薄切片设计 / fallback 链 / 不翻译 agent 输出 / 7 个生产语言 + 9 个 best-effort

**更新 12 个现有概念页**：
- concepts/prompt-caching-optimization.md      — + 5m/1h 双档 TTL + 跨 session 1h cache
- concepts/messaging-gateway-architecture.md   — + 22 平台清单 + 5 插件平台细节
- concepts/multi-agent-architecture.md         — + 5 种机制（含 Kanban + goal loop）+ orchestrator/max_spawn_depth/file_state 协调
- concepts/browser-tool-architecture.md        — + 180× CDP 持久 WS + Chromium 自动启动 + SSRF floor
- concepts/cli-architecture.md                 — + Ink TUI 双前端 + 新子命令 + 新斜杠命令
- concepts/cron-scheduling.md                  — + no_agent 模式 + webhook 直送 + 注入扫描
- concepts/security-defense-system.md          — + 20 个 P0 (v0.13.0 + v0.14.0) + OSV 扫描 + [all] 减肥
- concepts/session-search-and-sessiondb.md     — + state.db 唯一权威 + JSONL 退役 commit 列表
- concepts/gateway-session-management.md       — + 自动续约 + X-Hermes-Session-Key
- concepts/smart-model-routing.md              — + 与 ProviderProfile 协作 + 全部新增 provider 列表
- concepts/voice-mode-architecture.md          — + TTS provider registry + Piper / xAI custom voices
- concepts/provider-transport-architecture.md  — + 与 ProviderProfile 分工说明 + HEAD 期 transport 清单
- concepts/skills-system-architecture.md       — + Background Review v2 + Curator 9 子命令 + 新 bundled skills
- concepts/web-tools-architecture.md           — + 按 capability 拆 backend + SearXNG + xAI Web Search
- concepts/terminal-backends.md                — + 自适应 poll −195ms + Windows 修复

**README + log**：
- README.md：版本 v2026.4.23 → v0.14.0 (v2026.5.16) + 21 post-release commits；页数 37 → 43；最后更新 2026-04-29 → 2026-05-20
- changelog/2026-05-20-update.md：新增完整更新日志，13 个主题节，每节带源码 path:line 验证
- log.md：本条目

**验证方式**：源码 clone 到 `/home/user/hermes-agent`，HEAD `31a010010`，每条结论用 grep / 行号 / 头部 docstring 复核；release notes (`RELEASE_v0.11.0.md` ~ `RELEASE_v0.14.0.md`) 用作发布主题索引。
- index.md 更新为 43 页
