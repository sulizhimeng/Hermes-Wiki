---
title: i18n 国际化与多语言架构
created: 2026-05-20
updated: 2026-05-20
type: concept
tags: [i18n, locales, ux]
source_files:
  - agent/i18n.py
  - locales/
verified_against: hermes-agent HEAD (2026-05-20)
---

# i18n 国际化（v0.13.0）

`agent/i18n.py` + `locales/*.yaml` 引入了 **薄切片**国际化 —— 只翻译"hermes 自己显示给用户的静态字符串"，不动 agent 输出、日志、stack trace。

## 1. 设计原则：薄切片

源码注释（`agent/i18n.py:1-20`）直引：

> Scope (thin slice, by design): only the highest-impact static strings shown to the user by Hermes itself —— approval prompts, a handful of gateway slash command replies, restart-drain notices. **Agent-generated output, log lines, error tracebacks, tool outputs, and slash-command descriptions all stay in English.**

**为什么薄？**

| 不翻译 | 理由 |
|--------|------|
| Agent 输出 | LLM 根据用户语言自适应，不由 hermes 决定 |
| 日志行 | 给开发者看，英文便于 grep |
| Tool 输出 | 同上，且大多是结构化数据 |
| 错误 traceback | Python 自己的，hermes 不要拦 |
| 斜杠命令描述 | agent 也要读，混语言反而困扰模型 |

翻译的：

- approval prompts（用户每天看 N 次的危险命令确认）
- 几条 gateway 斜杠命令回复
- restart-drain 通知（"网关 30 秒后重启，X 个会话进入 drain"）

## 2. 解析顺序

```python
# agent/i18n.py
def t(key: str, *, lang: Optional[str] = None, **fmt_args) -> str:
    """Translate static string identified by `key`.

    Language resolution order:
        1. Explicit ``lang=`` argument (tests / 临时覆盖)
        2. ``HERMES_LANGUAGE`` env var
        3. ``display.language`` in config.yaml
        4. ``"en"`` (baseline)
    """
```

Catalog 是**扁平 dict 键路径**（如 `approval.choose_long`、`gateway.draining`），不是嵌套树 —— 简单且查找 O(1)。

## 3. 文件布局

```
locales/
├── en.yaml      # baseline，hermes 不存在的 key 永远从这里 fallback
├── zh.yaml      # 简体中文
├── zh-hant.yaml # 繁体中文（v0.13.0+）
├── ja.yaml
├── de.yaml
├── es.yaml
├── fr.yaml
├── tr.yaml
├── uk.yaml      # 乌克兰语
├── af.yaml      # 南非荷语
├── ga.yaml      # 爱尔兰语
├── hu.yaml      # 匈牙利语
├── it.yaml
├── ko.yaml
├── pt.yaml      # 葡萄牙语
└── ru.yaml      # 俄语
```

HEAD 期 **16 个 YAML 文件**。release notes 强调的"7 个 first-class 生产语言"是有完整覆盖 + 测试 + 文档 site 翻译的子集：**zh / ja / de / es / fr / uk / tr**。其余 9 个属于社区贡献的 best-effort 翻译。

文档站（`website/`）同步增加了 **zh-Hans 翻译**（v0.13.0 PR #20329）。

## 4. Fallback 链

```python
def t(key, lang=None, **fmt_args):
    # 1. 在 user lang catalog 找
    # 2. 找不到就在 en catalog 找
    # 3. 还找不到就返回 key 本身（如 "approval.choose"）
    # 4. 任何步骤抛异常都返回 key 本身 —— 绝不 crash agent
```

> 这是 "**broken catalog never crashes the agent**" 的关键不变量 —— i18n 是显示层，永远不能成为可用性故障点。

## 5. 实际用法

```python
from agent.i18n import t

# 简单字符串
print(t("approval.choose_long"))

# 带参数
print(t("gateway.draining", count=3))
# zh.yaml:  gateway:
#             draining: "网关 30 秒后重启 — {count} 个会话进入 drain"

# 显式覆盖语言
print(t("approval.choose_long", lang="zh"))
```

## 6. 与现有系统的关系

| 系统 | 是否走 i18n |
|------|-------------|
| approval / 危险命令 prompt | **是** |
| gateway slash command 回复（部分） | **是** |
| restart drain 通知 | **是** |
| TUI 的固定 label（如 "Status"、"Press 'q' to quit"） | TUI 自己的 locale 子集 |
| Web Dashboard | dashboard 独立 i18n（v0.11.0 起，英 + 中） |
| LLM 系统提示 | **否**（英文，保持模型最佳行为） |
| skill 描述 | **否**（agent 也读） |
| tool 输出 | **否** |
| 错误 / 日志 | **否** |

---

## 7. 不变量

- catalog 永远 **flat dict** + 点号路径键。
- 缺 key 永远 fallback 英文，缺英文永远 fallback key 字符串本身。
- 解析顺序固定：`lang=` → `HERMES_LANGUAGE` → config → `"en"`。
- **不翻译** agent 自生成内容；只翻译 hermes 框架自己说的话。
- 翻译失败永不 crash。

## 8. 验证

```
agent/i18n.py:1-30          docstring 明确"薄切片"原则 + 解析顺序
locales/*.yaml              16 个语言文件
website/docs/i18n/          文档站 zh-Hans 翻译路径
```
