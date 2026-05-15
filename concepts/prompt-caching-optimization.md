---
title: Prompt Caching 优化架构
created: 2026-04-07
updated: 2026-05-15
type: concept
tags: [architecture, module, performance, cost-optimization, anthropic]
sources: [agent/prompt_caching.py, run_agent.py]
---

# Prompt Caching — Anthropic 缓存优化架构

## 概述

Prompt Caching 位于 `agent/prompt_caching.py`（约 80 行），实现 **Anthropic `system_and_3` 缓存策略**，在单个会话的多轮对话中减少约 75% 的输入 token 成本。

核心理念：**单一布局，最多 4 个 cache_control 断点 — 系统提示 + 最后 3 条非系统消息。**

> **演进备注（2026-05）**：曾短暂引入过一个跨会话 1 小时长效前缀缓存布局（`feat(prompt-cache)` #23828），把工具数组 + 稳定系统前缀单独标 `ttl=1h`、易变内容（时间戳 / memory / USER profile）拆到尾部独立块。但实测发现易变层每轮都在变，导致**系统消息字节在会话中途变化**，反而打穿了上游缓存（OpenRouter / Nous Portal / Anthropic）。该长效布局已被 `fix(cache)` #24778 **整体回退**，回到下文描述的单一 `system_and_3` 布局，并确立不变量：**系统提示在一个会话内字节级静态（byte-static）** —— 见下文「系统提示字节静态不变量」。

## 架构原理

### system_and_3 策略

Anthropic 的 prompt cache 允许在消息中标记 `cache_control` 断点。断点前的内容会被缓存，后续请求命中缓存时只收取极低的 cache_read 费用（约为正常费用的 10%）。

Anthropic 限制**最多 4 个断点**，Hermes 的分配策略：

| 断点 | 位置 | 缓存内容 | 稳定性 |
|---|---|---|---|
| 1 | 系统提示 | 身份 + 平台提示 + 技能索引 | 最高（跨所有轮次不变） |
| 2 | 倒数第 3 条消息 | 早期对话内容 | 高（前 2 轮不变） |
| 3 | 倒数第 2 条消息 | 中期对话内容 | 中（前 1 轮不变） |
| 4 | 最后 1 条消息 | 最近的对话内容 | 低（每轮滚动） |

### 滚动窗口机制

```
轮次 1: [系统提示★] [用户1★] [助手1] [助手2]
                        ↑断点2 ↑断点3  ↑断点4

轮次 2: [系统提示★] [用户1] [助手1★] [用户2★] [助手2★]
                        ↑新断点2 ↑新断点3 ↑新断点4

轮次 3: [系统提示★] [用户1] [助手1] [用户2] [助手2★] [用户3★] [助手3★]
                                                      ↑新断点2 ↑新断点3 ↑新断点4
```

★ = cache_control 标记。每次新请求时，断点窗口向后滚动。

## 核心组件

### 1. cache_control 标记注入

```python
def _apply_cache_marker(msg, cache_marker, native_anthropic=False):
    """
    处理所有消息格式变体:
    
    1. tool 角色 → 只在 native_anthropic 模式下标记
    2. 空内容 → 直接在消息级别标记
    3. 字符串内容 → 转换为 [{"type": "text", "text": ..., "cache_control": ...}]
    4. 列表内容 → 在最后一个元素上添加 cache_control
    """
```

**设计考量**：Anthropic API 接受多种消息格式（字符串、对象列表、工具结果），`_apply_cache_marker` 统一处理所有格式。

### 2. 主函数

```python
def apply_anthropic_cache_control(
    api_messages,
    cache_ttl="5m",        # 缓存 TTL: 5分钟 或 1小时
    native_anthropic=False # 是否使用原生 Anthropic 格式
):
    """
    1. 深拷贝消息（不修改原始数据）
    2. 用 _build_marker(ttl) 创建 marker:
       {"type": "ephemeral"} 或 {"type": "ephemeral", "ttl": "1h"}
    3. 系统提示添加断点（如果是第一条消息）
    4. 从后向前找最后 3 条非系统消息（断点预算 4 - 已用），添加断点
    5. 返回标记后的消息列表
    """
```

`_build_marker(ttl)` 是独立的辅助函数：默认返回 `{"type": "ephemeral"}`，仅当 `ttl == "1h"` 时附加 `"ttl": "1h"`。模块只有这一种布局（`apply_anthropic_cache_control`）；早期版本曾有的 `apply_anthropic_cache_control_long_lived()` / `mark_tools_for_long_lived_cache()` 已随 #24778 删除。

### 3. 不同角色的处理

| 角色 | 缓存策略 |
|---|---|
| system | 始终标记（最稳定的缓存点） |
| tool | 仅 native_anthropic 模式下在消息级别标记 |
| assistant/user | 在 content 的最后一个元素上标记 |

## TTL 配置

```python
marker = {"type": "ephemeral"}         # 默认: 5 分钟 TTL
marker = {"type": "ephemeral", "ttl": "1h"}  # 1 小时 TTL
```

TTL 由 `config.yaml` 的 `prompt_caching.cache_ttl` 控制（默认 `"5m"`），只接受 `"5m"` 或 `"1h"` 两个 Anthropic 支持的档位，其他值被忽略并保持 `"5m"`：

```yaml
# config.yaml
prompt_caching:
  cache_ttl: "5m"   # 长会话且轮次间有停顿可设为 "1h"
```

**使用场景**：
- **5m（默认）**：适合快速连续对话，缓存命中率高
- **1h**：适合长时间对话间隔，容忍更高的缓存未命中

> 此 `cache_ttl` 是**会话内滚动窗口**的 TTL。它并不提供跨会话的前缀复用——短暂存在过的 `long_lived_prefix` / `long_lived_ttl` 配置键已随 #24778 移除。

## 系统提示字节静态不变量

Hermes 不变量：**系统提示在一个会话内必须字节级静态（byte-static）**。`run_agent.py` 中系统提示在会话首轮构建一次，缓存在 `self._cached_system_prompt` 上，之后每一轮**原样逐字节重放**。

- 续接会话（gateway 为每条消息新建 `AIAgent`）时，从 session DB 读回**已存储的系统提示**，而不是重建——重建会把模型自己写入的 memory 变化重新读进来，产生不同的系统提示从而打穿前缀缓存。
- 系统提示作为**单个 content 字符串**发送，保证字节稳定。

之所以确立这条不变量，是因为 #24778 诊断出：旧的长效前缀布局把系统提示拆成「稳定 / 上下文 / 易变」三块并每轮重新派生，易变块（时间戳 + memory 快照 + USER profile）每轮都变，导致 8 轮对话中系统块在分钟边界处 sha 翻转、当轮 `cached_tokens` 掉到 0。回退到单块布局后，会话内缓存在每个 provider 上才真正稳定生效。

## 成本效益

假设系统提示 2000 tokens，每次对话平均 5000 tokens：

| 场景 | 无缓存成本 | 缓存命中成本 | 节省 |
|---|---|---|---|
| 单轮（系统提示 + 1 条消息） | ~7000 tokens × 价格 | ~2000 tokens × cache_read + 5000 × 正常 | ~70% |
| 10 轮对话 | 10 × 7000 = 70K tokens | ~2000 × cache_read + (70K-2000) × 正常 | ~75% |
| 50 轮对话 | 50 × 7000 = 350K tokens | ~2000 × cache_read + (350K-2000) × 正常 | ~85% |

## 设计优越性

### 对比无缓存方案

| 维度 | 无缓存 | Prompt Caching |
|---|---|---|
| 系统提示成本 | 每次付费 | 仅首次付费 |
| 早期对话成本 | 每次付费 | 命中时仅付 cache_read |
| 延迟 | 无影响 | 缓存命中时降低 |
| 代码复杂度 | 低 | 约 80 行纯函数 |
| 适用场景 | 所有模型 | 仅 Claude / Qwen 等接受 cache_control 的端点 |

### 纯函数设计

```python
# 无类状态，无 AIAgent 依赖
# 输入消息列表 → 输出标记后的消息列表
# 深拷贝确保不修改原始数据
```

这使得缓存逻辑可以独立测试，且不影响主对话流程。

## 集成点与 Provider 适用范围

Prompt caching 在 `run_agent.py` 构建 API 请求时被调用：
1. 构建完整消息列表，把 `_cached_system_prompt` 作为单字符串 system 消息插到最前
2. 如果 `self._use_prompt_caching` 为真 → 调用 `apply_anthropic_cache_control(api_messages, cache_ttl=self._cache_ttl, native_anthropic=self._use_native_cache_layout)`
3. 兜底清理孤立 tool 结果后发送给 API

是否启用、用哪种布局由 `_anthropic_prompt_cache_policy()` 决定，返回 `(should_cache, use_native_layout)`：

| 端点 | should_cache | 布局 |
|---|---|---|
| 原生 Anthropic（含 OAuth 订阅） | True | native（标记打在 content 内层块上） |
| OpenRouter 上的 Claude | True | envelope（标记打在消息外层） |
| Nous Portal 上的 Claude | True | envelope（Portal 代理到 OpenRouter） |
| Nous Portal 上的 Qwen（如 qwen3.6-plus） | True | envelope |
| Alibaba / Qwen 系（OpenCode 等） | True | envelope |
| 其他 provider | False | 不缓存 |

> **Portal Qwen 的 TTL 限制（#24702）**：Nous Portal Qwen 最终代理到 Alibaba DashScope，其 Context Cache 只支持单一 5 分钟 ephemeral TTL，`ttl="1h"` 会被上游静默丢弃。因此 Portal Qwen 走标准 `system_and_3` 5m 布局——`cache_ttl="1h"` 对它无效。1h TTL 仅对 Anthropic / OpenRouter 的 Anthropic 路由真正生效。

## 与其他系统的关系

- [[smart-model-routing]] — 缓存成本信息来自 models.dev
- [[auxiliary-client-architecture]] — 辅助模型不使用 prompt caching
- [[context-compressor-architecture]] — 上下文压缩减少消息数量，间接影响缓存断点位置
