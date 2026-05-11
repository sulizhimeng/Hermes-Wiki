---
title: Prompt Caching 优化架构
created: 2026-04-07
updated: 2026-05-11
type: concept
tags: [architecture, module, performance, cost-optimization, anthropic]
sources: [agent/prompt_caching.py, run_agent.py]
---

# Prompt Caching — Anthropic 缓存优化架构

## 概述

Prompt Caching 位于 `agent/prompt_caching.py`，实现 **两套** Anthropic 缓存策略：

- **`system_and_3`**（默认）—— 4 个 cache_control 断点，**系统提示 + 最后 3 条非系统消息**，同一 TTL（5m 或 1h）。多轮对话减少约 75% 输入 token 成本
- **`prefix_and_2`**（v0.13.0 后续 #23828 引入，Claude on Anthropic / OpenRouter / Nous Portal）—— 4 个断点拆**两个 TTL 层**：`tools[-1]` (1h) + 稳定 system prefix (1h) + 末 2 条非 system message (5m)。**首轮新 session 输入成本砍 ~85–90%**

核心理念：**最多 4 个 cache_control 断点**。`system_and_3` 把它们全部放在末尾窗口；`prefix_and_2` 把 2 个搬到跨 session 共享的稳定前缀上，剩 2 个留给末尾滚动。

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
    2. 创建 marker: {"type": "ephemeral"} 或 {"type": "ephemeral", "ttl": "1h"}
    3. 系统提示添加断点（如果是第一条消息）
    4. 从后向前找最后 3 条非系统消息，添加断点
    5. 返回标记后的消息列表
    """
```

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

**使用场景**：
- **5m（默认）**：适合快速连续对话，缓存命中率高
- **1h**：适合长时间对话间隔，容忍更高的缓存未命中
- **可配置**：`prompt_caching.cache_ttl: 5m | 1h`（v0.12.0 引入，bursty session 保温更划算）

## `prefix_and_2` —— 跨 Session 1h 前缀缓存（v0.13.0 后续）

源码（`agent/prompt_caching.py:10-22`）：

> 4 breakpoints split across two TTL tiers — `tools[-1]` (1h) + stable system prefix (1h) + last 2 non-system messages (5m). The **long-lived prefix is byte-stable across sessions** for a given user config, so every fresh session reads the cached system+tools instead of re-paying for them. Within-session rolling window shrinks from 3 messages to 2 to free the breakpoint budget.

### 关键设计：byte-stable prefix

要让 prefix 跨 session 共享，必须**逐字节稳定**。Hermes 的做法：
- 把不稳定的部分（memory / USER profile / 当前时间戳 / session_id）**全部移到 block[0] 之后**
- block[0] 只放：tools schema + 系统身份 + 平台提示 + skills 索引（稳定到只有用户改 config 才会变）
- volatile suffix 放到下一个 block，不享受 1h prefix cache，但享受 5m 滚动窗口

### 调度顺序

Anthropic prefix-cache 的物理顺序是 `tools → system → messages`。标记位置：

```
tools[..., tools[-1]★1h]
 → system[block[0]★1h, block[1+]volatile]
 → messages[..., msg[-2]★5m, msg[-1]★5m]
```

### 触发条件

- provider 是 Claude（Anthropic native / OpenRouter / Nous Portal）
- session 内有过任一轮 tool 调用（保证 tools array 非空）
- 1h 内有相同 user config 的请求

### 成本效益（实测，#23828 PR 描述）

- 首轮新 session 输入成本 **↓ 85–90%**
- Tools array (~13k tokens 默认 toolset) + 稳定 system prefix (~5–8k tokens) = ~20k tokens 走 cache_read（约正常 10%）
- 之前 `system_and_3` 跨 session 完全冷启动，每个新 session 都全价付这 20k

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
| 代码复杂度 | 低 | 72 行纯函数 |
| 适用场景 | 所有模型 | 仅 Anthropic 模型 |

### 纯函数设计

```python
# 无类状态，无 AIAgent 依赖
# 输入消息列表 → 输出标记后的消息列表
# 深拷贝确保不修改原始数据
```

这使得缓存逻辑可以独立测试，且不影响主对话流程。

## 集成点

Prompt caching 在 `run_agent.py` 的 `_build_api_kwargs()` 中被调用：
1. 构建完整消息列表
2. 如果当前 provider 是 Anthropic → 调用 `apply_anthropic_cache_control()`
3. 如果 `developer_role` 需要切换 → 将 system 消息转为 developer 角色
4. 发送给 API

## 与其他系统的关系

- [[smart-model-routing]] — 缓存成本信息来自 models.dev
- [[auxiliary-client-architecture]] — 辅助模型不使用 prompt caching
- [[context-compressor-architecture]] — 上下文压缩减少消息数量，间接影响缓存断点位置
