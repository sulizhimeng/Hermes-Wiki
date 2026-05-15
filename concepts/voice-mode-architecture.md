---
title: 语音模式架构
created: 2026-04-10
updated: 2026-05-15
type: concept
tags: [voice, stt, tts, architecture]
sources: [tools/voice_mode.py, tools/tts_tool.py, tools/transcription_tools.py, cli.py]
---

# 语音模式架构

## 概述

Hermes 支持 Push-to-talk 语音交互：用户按键录音 → STT 转文字 → LLM 处理 → TTS 语音播报回复。整个链路在 CLI 中完成，依赖可选的音频库。

## 依赖

```bash
pip install sounddevice numpy   # 或
pip install hermes-agent[voice]
```

音频库**按需懒加载**，不装也不影响文本模式。在无音频设备的环境（SSH、Docker、WSL）中自动检测并禁用。

## 流程

```text
用户按 Ctrl+B 开始录音
    ↓
sounddevice 采集音频 → WAV 临时文件
    ↓
再按 Ctrl+B 停止录音
    ↓
STT 转文字（5 个 Provider 可选）:
  - local: faster-whisper（本地，无需 API Key，默认）
  - groq: Whisper via Groq（免费额度）
  - openai: Whisper via OpenAI
  - mistral: Voxtral Transcribe via Mistral
  - xai: Grok STT via xAI
    ↓
转录文本作为用户消息发送给 LLM
    ↓
LLM 回复（自动注入简洁指令："respond concisely, 2-3 sentences max"）
    ↓
TTS 语音播报（10 个内置 Provider + 自定义 command Provider）:
  - edge（Edge TTS）
  - elevenlabs（流式，边生成边播放）
  - openai（OpenAI TTS）
  - gemini（Google Gemini TTS）
  - xai（xAI TTS）
  - minimax（MiniMax TTS）
  - mistral（Mistral TTS）
  - neutts（自托管）
  - kittentts（本地 CPU）
  - piper（本地神经 TTS）
```

## STT 配置

```yaml
# config.yaml
stt:
  provider: local   # local | groq | openai | mistral | xai
  model: base       # faster-whisper 模型大小（base ~150MB，首次自动下载）
```

未显式指定 provider 时按 `local > groq > openai > xai` 自动探测（`mistral` 因 `mistralai` PyPI 包于 2026-05-12 被隔离暂时跳过）。

```bash
# .env
GROQ_API_KEY=...              # Groq Whisper（免费）
VOICE_TOOLS_OPENAI_KEY=...    # OpenAI Whisper
MISTRAL_API_KEY=...           # Mistral Voxtral Transcribe
XAI_API_KEY=...               # xAI Grok STT
```

## TTS 配置

TTS Provider 选择和语音设置通过 `tools/tts_tool.py` 管理，支持 ElevenLabs 的流式播报——LLM 生成一句就播一句，不用等完整回复。

### 内置 TTS Provider

`tools/tts_tool.py` 的 `BUILTIN_TTS_PROVIDERS` 当前包含 10 个内置 provider：

| Provider | 说明 |
|----------|------|
| `edge` | Edge TTS |
| `elevenlabs` | ElevenLabs，支持流式播报 |
| `openai` | OpenAI TTS |
| `gemini` | Google Gemini TTS（通过 Gemini API） |
| `xai` | xAI TTS |
| `minimax` | MiniMax TTS（默认模型 `speech-02`，支持 GroupId） |
| `mistral` | Mistral TTS |
| `neutts` | NeuTTS（自托管） |
| `kittentts` | KittenTTS，本地 CPU 运行，无需 GPU 和 API key，默认模型 `KittenML/kitten-tts-nano-0.8-int8`（25MB），默认声音 `Jasper` |
| `piper` | **Piper（v2026-05-15+ 新增）**，来自 Home Assistant 项目的快速本地神经 TTS（OHF-Voice/piper1-gpl），支持 44 种语言，零 API key |

部分云端 provider 也可通过 Nous Tool Gateway 统一访问（无需自备 API key）。

### Piper 本地 TTS（v2026-05-15+）

`piper` 作为原生内置 provider 加入（closes #8508 / #17885），与 `edge`/`neutts`/`kittentts` 并列，可通过 `hermes tools` 一键安装：

- 懒加载导入 `_import_piper()`
- 模块级 voice 缓存以 `(model_path, use_cuda)` 为 key——切换声音不会使旧缓存失效
- `_resolve_piper_voice_path()` 接受绝对 `.onnx` 路径或声音名（首次使用经 `python -m piper.download_voices` 自动下载）
- voice 缓存位于 `~/.hermes/cache/piper-voices/`（profile-aware）
- 可选 `SynthesisConfig` 参数：`length_scale`、`noise_scale`、`noise_w_scale`、`volume`、`normalize_audio`、`use_cuda`
- 输出 WAV 后经 ffmpeg 转换（与 neutts/kittentts 一致），ffmpeg 存在时支持 Telegram 语音气泡

### command 类型 Provider 注册表（v2026-05-15+）

`tts.providers.<name>` 注册表（#17843）让用户无需添加引擎专属 Python 代码即可接入任意本地/外部 TTS CLI：

```yaml
tts:
  provider: piper-en
  providers:
    piper-en:
      type: command
      command: 'piper -m ~/model.onnx -f {output_path} < {input_path}'
      output_format: wav
```

- 占位符：`{input_path}`、`{text_path}`、`{output_path}`、`{format}`、`{voice}`、`{model}`、`{speed}`；`{{` / `}}` 表示字面花括号
- `command:` 已设置时 `type: command` 为默认
- **内置 provider 名永远优先**——`tts.providers.openai` 等条目不能遮蔽原生内置 provider

### STT Provider（5 个）

`tools/transcription_tools.py` 提供 5 个 STT provider：

| Provider | 说明 |
|----------|------|
| `local` | faster-whisper 本地运行，默认，无需 API key，自动下载模型 |
| `groq` | Groq Whisper API（免费额度），需 `GROQ_API_KEY` |
| `openai` | OpenAI Whisper API，需 `VOICE_TOOLS_OPENAI_KEY` |
| `mistral` | Mistral Voxtral Transcribe API，需 `MISTRAL_API_KEY`（因 `mistralai` 包于 2026-05-12 被隔离暂时禁用） |
| `xai` | xAI Grok STT API，需 `XAI_API_KEY`，支持 ITN（Inverse Text Normalization）+ 可选 diarization，21 种语言 |

## 语音模式特殊行为

- LLM 收到语音输入时，系统自动注入前缀指令要求简短回复
- 该前缀仅用于 API 调用，**不持久化到会话历史**（通过 `persist_user_message` 参数保存原始转录文本）
- 持续语音模式下遇到持久错误（如 429）会自动停止，防止错误 → 录音 → 错误的死循环

## 相关页面

- [[cli-architecture]] — CLI 中的语音模式集成
- [[auxiliary-client-architecture]] — STT/TTS 使用 auxiliary 模型配置

## 关键源码

| 文件 | 职责 |
|------|------|
| `tools/voice_mode.py`（812 行）| 录音、STT 调度、音频播放 |
| `tools/tts_tool.py`（983 行）| TTS Provider 路由、流式播报 |
| `tools/transcription_tools.py` | STT Provider 统一接口 |
| `cli.py` | Push-to-talk 键绑定（Ctrl+B） |
