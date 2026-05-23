# Wiki Index

> 内容目录。每个 wiki 页面按类型列出，附一行摘要。
> 查询前先读此文件找到相关页面。
> Last updated: 2026-05-23 | Total pages: 45 concepts + 2 entities + 27 changelogs | Tracking: v0.14.0 (hermes-agent/master HEAD `874c2b1`)

## Entities

- [[aiagent-class]] — 核心对话循环类，管理 LLM 交互和工具调用
- [[memorystore-class]] — 记忆系统核心类，管理 MEMORY.md 和 USER.md

## Concepts

### 核心架构
- [[tool-registry-architecture]] — 中央工具注册系统，声明式注册+集中调度，循环导入安全
- [[auxiliary-client-architecture]] — 辅助 LLM 客户端路由器，多 provider 解析链+适配器模式+自动降级
- [[browser-tool-architecture]] — 多后端浏览器自动化，accessibility tree 文本表示+三层安全防护+并发隔离
- [[web-tools-architecture]] — Web 搜索/提取/爬取 7 个 plugin 后端，按 capability 独立选 provider
- [[skills-system-architecture]] — 渐进式披露 + Curator 后台维护（archive/prune/list-archived）
- [[memory-system-architecture]] — 冻结快照模式、原子写入、安全扫描
- [[agent-loop-and-prompt-assembly]] — Agent 循环、系统提示构建、平台提示、执行指导
- [[skills-and-memory-interaction]] — Skills 与 Memory 的互补关系和决策树
- [[toolsets-system]] — 工具分组系统、递归解析、24 个 hermes-* 工具集
- [[session-search-and-sessiondb]] — FTS5 搜索 + LLM 摘要的跨会话回忆
- [[provider-transport-architecture]] — Provider Transport ABC（Anthropic / Chat / Responses / Bedrock）
- [[provider-plugin-system]] — **新** ProviderProfile ABC + 30 个 plugin（v0.13.0+）

### 工具与能力
- [[code-execution-sandbox]] — execute_code 沙箱，7 工具限制+UDS/File RPC 两种通信模式
- [[context-references]] — @file/@folder/@diff/@url/@git 引用系统，安全沙箱+注入量限制
- [[voice-mode-architecture]] — Push-to-talk 语音交互，STT（3 Provider）+ TTS（5 Provider）
- [[lsp-integration]] — **新** LSP 语义诊断集成（v0.14），agent/lsp/ 11 modules + file-mutation footer 三层后写校验
- [[checkpoints-architecture]] — Checkpoint Manager v2，透明文件系统快照（v0.13.0）
- [[i18n-and-locales]] — i18n 国际化（v0.13.0），薄切片本地化（仅 hermes 自身静态消息）

### 性能与优化
- [[parallel-tool-execution]] — 智能并发安全检测，三层分类 + 路径冲突检测
- [[prompt-caching-optimization]] — 冻结快照保护 prefix cache，75% 成本节省
- [[fuzzy-matching-engine]] — 8 策略链模糊匹配
- [[smart-model-routing]] — 智能模型路由，10级上下文长度解析链
- [[large-tool-result-handling]] — 三层溢出防护

### 安全与可靠性
- [[security-defense-system]] — 5 层防御体系，20 个 P0 闭合（v0.13.0+v0.14.0），100+ 威胁模式
- [[interrupt-and-fault-tolerance]] — 中断传播、凭证池轮换、Fallback 模型链
- [[credential-pool-and-isolation]] — 多密钥自动轮换、Profile 隔离
- [[tool-loop-guardrails]] — 工具调用循环护栏（v0.12.0），exact failure / same-tool failure / idempotent no-progress 三维检测

### 多 Agent
- [[multi-agent-architecture]] — 5 种 Agent 协作机制（delegate/MoA/Background Review/send_message/**Kanban**）
- [[kanban-multi-agent-board]] — 可持久化任务板，SQLite + 心跳 + 熔断 + 幻觉拦截（v0.13+）
- [[goal-and-ralph-loop]] — `/goal` + `/subgoal` 持续推进系统（v0.13/v0.14）

### 平台与扩展
- [[cli-architecture]] — 经典 CLI + Ink TUI 双前端、斜杠命令补全、Skin 引擎
- [[configuration-and-profiles]] — 分层配置、Profile 隔离、自动迁移
- [[hook-system-architecture]] — Hook 系统（Gateway Hooks + Plugin System），事件驱动+工具注册+上下文注入
- [[mcp-and-plugins]] — MCP 集成、插件钩子系统、OAuth 支持
- [[terminal-backends]] — 7 种终端后端（含 Vercel Sandbox）、环境抽象、持久化 Shell
- [[cron-scheduling]] — 内置调度器、自然语言调度、多平台投递、no_agent watchdog
- [[trajectory-and-data-generation]] — 轨迹保存、批量运行器、RL 训练环境
- [[prompt-builder-architecture]] — 系统提示模块化组装，注入防护+技能缓存+模型特定指导
- [[context-compressor-architecture]] — 自动上下文压缩，结构化摘要+迭代更新+工具对完整性保障
- [[model-tools-dispatch]] — 工具编排与调度，异步桥接+动态 schema 调整+参数类型强制
- [[gateway-session-management]] — 网关会话管理，多平台会话隔离+PII 脱敏+重置策略
- [[messaging-gateway-architecture]] — 22 平台统一网关（含 Teams/LINE/SimpleX/Google Chat 插件平台）
- [[hermes-proxy]] — OAuth 订阅 → OpenAI 兼容本地代理（v0.14）
- [[skin-engine]] — YAML 驱动的皮肤/主题系统
- [[worktree-isolation]] — Git Worktree 并行隔离模式

### 更新日志（按时间倒序）
- [[2026-05-23-update]] — 49 commits daily delta（Nous Portal one-shot、Kanban DB 抗污染、Memory drift guard、审批沉默契约、TLS FD 三层防御、Plugin RCE 第二段、Telegram/WhatsApp/QQBot）
- [[2026-05-22-update]] — 2,578 commits（v0.12.0 → v0.14.0）
- [[2026-05-20-update]] — v0.14.0（~2,480 commits）
- [[2026-05-19-update]] — 2,418 commits（v2026.4.23 → v2026.5.16）
- [[2026-05-18-update]] — v0.14.0（285 commits）
- [[2026-05-17-update]] — v2026.5.16（988 commits）
- [[2026-05-16-update]] — v2026.5.16（2,890 commits since v2026.4.23）
- [[2026-05-15-update]] — v0.13.0（579 commits）
- [[2026-05-14-update]] — 1,533 commits（v0.12.0 + v0.13.0 + post-release）
- [[2026-05-13-update]] — v0.12.0 → v0.13.0（595 commits）
- [[2026-05-12-update]] — v0.12.0 + v0.13.0（~2,015 commits）
- [[2026-05-11-update]] — v0.12.0 + v0.13.0 + post-release
- [[2026-05-10-update]] — v0.12.0 + v0.13.0（1,960 commits）
- [[2026-05-09-update]] — v0.13.0 (v2026.5.7) + post-release（1,100 commits）
- [[2026-05-07-update]] — v0.12.0 + v0.13.0（1,960 commits）
- [[2026-05-07-v0.13.0]] — v0.13.0 release notes
- [[2026-05-06-update]] — v0.12.0（211 commits）
- [[2026-05-05-update]] — v0.12.0（99 commits）
- [[2026-05-04-update]] — v0.12.0 (v2026.4.30) + 71 post-release commits
- [[2026-05-02-update]] — v2026.4.30（1,338 commits）— The Curator Release
- [[2026-04-30-update]] — v2026.4.30
- [[2026-04-30-v0.12.0]] — v0.12.0 release notes
- [[2026-04-29-update]] — 182 commits (v2026.4.23)
- [[2026-04-18-update]] — 410 commits post-v0.10.0
- [[2026-04-17-update]] — 641 commits (v0.10.0)
- [[2026-04-10-update]] — 293 commits
- [[2026-04-09-update]] — 59 commits
