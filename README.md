# 尘墨·视频知识采集 (Chenmo Video KB)

> **From video to knowledge — a fully automated, structured acquisition pipeline.**

Chenmo is an [OpenClaw](https://openclaw.ai) skill that transforms videos from any platform into searchable, reusable, structured knowledge documents. It doesn't just download files — it **extracts meaning**: audio → transcription → AI distillation → categorized archive, all in one pipeline.

---

## 🧠 Why Chenmo?

The internet is drowning in video content — tutorials, talks, deep dives — but video is inherently **unsearchable**. You can't Ctrl+F a video. You can't skim it at 10x speed without losing context. And rebuilding knowledge from the same video months later means re-watching the entire thing.

**Chenmo solves this**: it converts video into structured Markdown knowledge bases with TL;DR summaries, timestamped key moments, actionable takeaways, and automatic classification — so you read 3 minutes instead of watching 30.

---

## ✨ What It Does

| Capability | Description |
|---|---|
| 🎬 **Multi-platform** | Bilibili, YouTube, and any yt-dlp supported platform |
| 🔊 **Audio extraction** | Downloads audio streams via Bilibili dash API or yt-dlp |
| 📝 **Transcription** | Built-in subtitles first (Bilibili API + YouTube), falls back to OpenAI Whisper (base model) |
| 🧠 **AI distillation** | Extracts TL;DR, core insights, actionable techniques, gotchas, and timestamped key moments |
| 📂 **Auto-classification** | Categorizes into 10 topic areas (AI tools, image generation, programming, design, etc.) |
| 📄 **Structured archive** | Generates full Markdown docs with metadata cards, embedded players, cover images, and knowledge summaries |
| 🔄 **Incremental updates** | Detects existing archives and appends only new episodes/parts |
| ⚡ **Parallel processing** | Multi-episode series automatically split across sub-agents for parallel transcription + distillation |
| 🛡️ **28 engineering safeguards** | Error code mapping, subtitle JSON compatibility, fallback URL retries, disk space checks, timeout detection, and more |

---

## 🎯 Use Cases

- **Learning a new framework** — Archive a 20-episode tutorial series into a single searchable knowledge base, find any concept in seconds
- **Conference talks & deep dives** — Capture the key insights from a 2-hour keynote without re-watching
- **Research & reference** — Build a personal video wiki: every technique, config, or insight is timestamped and searchable
- **Content creation** — Extract reference material, quotes, and techniques from creator videos
- **Language learning** — Transcribe foreign-language videos (Chinese/English/Japanese/Korean supported), keep original + Chinese summary

---

## 🔧 How It Works (11-Step Pipeline)

```
Video URL → Platform Detection → Metadata Extraction → Language Detection
  → Subtitle/Transcription → AI Distillation → Auto-Classification
  → Structured Archive → Cleanup & Report
```

1. **Dependency check** — Auto-installs yt-dlp, ffmpeg, Whisper if missing; pre-downloads Whisper base model (~140MB)
2. **Network probe** — Tests reachability of target platforms before starting
3. **Deduplication** — Checks existing archives to avoid duplicate work
4. **Platform routing** — Bilibili API for dash audio streams, yt-dlp for everything else
5. **Metadata extraction** — Title, creator, cover, play count, publish date, duration, episode list (with full Bilibili error code mapping)
6. **Language detection** — Auto-detects video language, matches correct Whisper model (Chinese/English/Japanese/Korean/Auto)
7. **Transcription** — Built-in subtitles first (Bilibili API with multi-format JSON parsing); Whisper transcription as fallback with 30-minute timeout guard
8. **AI distillation** — Per-section: TL;DR, 3 core takeaways, key concepts, actionable techniques, gotchas, timestamped highlights
9. **Auto-classification** — Routes to 10 categories (AI tools, image gen, video/audio, writing, programming, design, reading, productivity, etc.)
10. **Archive generation** — Markdown doc with metadata card, embedded video player (iframe + fallback link), cover image, full knowledge summary
11. **Cleanup & timing** — Reports total processing time, cleans temp files, preserves error logs if any

### 📊 Smart Duration Handling

| Duration | Strategy |
|---|---|
| < 60s | Index only (metadata + cover), skip distillation |
| 60s – 3min | Full pipeline, condensed extraction (1–2 points per section) |
| > 3min | Full pipeline with comprehensive distillation |

---

## 🧰 Tech Stack

| Component | Role |
|---|---|
| **yt-dlp** | Multi-platform video/audio download (Bilibili, YouTube, Douyin, Vimeo, Twitter, etc.) |
| **ffmpeg** | Audio format conversion (m4s → mp3) |
| **OpenAI Whisper (base)** | Speech-to-text for videos without built-in subtitles; multi-language support |
| **Bilibili Player API** | Metadata, dash audio streams, subtitle data, with full error code handling |
| **OpenClaw sessions_spawn** | Parallel sub-agent scheduling for multi-episode series |

---

## 🚫 Safety Boundaries

Chenmo explicitly **refuses** to process:
- Paywalled or members-only content
- Copyright-protected material with explicit "no download/redistribution" notices
- Private videos involving personal privacy
- Adult or prohibited content

---

## 🚀 Getting Started

### Prerequisites

```bash
# yt-dlp — multi-platform video download
pip install yt-dlp

# ffmpeg — audio/video conversion
brew install ffmpeg

# OpenAI Whisper — speech transcription
pip install openai-whisper
```

### Installation

```bash
# Install as an OpenClaw skill
npx skills add chenmo-video-kb
```

### Usage

Simply send a video URL to your OpenClaw agent:

> "帮我提炼这个视频 https://www.bilibili.com/video/BV1xx411c7mD"

> "Archive this tutorial series: https://www.youtube.com/watch?v=..."

Chenmo automatically activates and runs the full pipeline.

---

## 🏷️ Branding

All outputs follow the **尘墨 (Chenmo)** brand standard:
- Archive filename prefix: `尘墨-`
- Temp file prefix: `cm_`
- Document footer: `*Generated by 尘墨·视频知识采集 Skill*`
- Standard completion report with timing, word count, and archive path

---

## 📄 License

MIT

---

*Built with ❤️ for the OpenClaw ecosystem.*
