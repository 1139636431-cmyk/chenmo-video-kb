# 尘墨·视频知识采集 (Chenmo Video KB)

<p align="center">
  <img src="https://img.shields.io/badge/version-1.0.0-blue" alt="version">
  <img src="https://img.shields.io/badge/platform-macOS%20%7C%20Linux-lightgrey" alt="platform">
  <img src="https://img.shields.io/badge/license-MIT-green" alt="license">
  <img src="https://img.shields.io/badge/OpenClaw-skill-orange" alt="OpenClaw skill">
</p>

> **From video to knowledge — a fully automated, structured acquisition pipeline.**

Chenmo is an [OpenClaw](https://openclaw.ai) skill that transforms videos from any platform into searchable, reusable, structured knowledge documents. It doesn't just download files — it **extracts meaning**: audio → transcription → AI distillation → categorized archive, all in one automated pipeline.

---

## 📑 Table of Contents

- [Why Chenmo?](#-why-chenmo)
- [What Sets It Apart](#-what-sets-it-apart)
- [Features](#-features)
- [Use Cases](#-use-cases)
- [Pipeline Overview](#-pipeline-overview)
- [Sample Output](#-sample-output)
- [Tech Stack](#-tech-stack)
- [Classification Categories](#-classification-categories)
- [Safety Boundaries](#-safety-boundaries)
- [Getting Started](#-getting-started)
- [FAQ](#-faq)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🧠 Why Chenmo?

The internet is drowning in video content — tutorials, conference talks, deep dives — but video is inherently **unsearchable**:

- ❌ You can't `Ctrl+F` a video
- ❌ You can't skim at 10x speed without losing context
- ❌ Rebuilding knowledge months later means re-watching the entire thing
- ❌ Tutorial series with 20+ episodes become impossible to navigate

**Chenmo solves this** by converting video into structured Markdown knowledge bases with TL;DR summaries, timestamped key moments, actionable takeaways, and automatic classification — so you **read 3 minutes instead of watching 30**.

---

## 🆚 What Sets It Apart

| Dimension | Traditional Downloaders | Chenmo Video KB |
|---|---|---|
| **Independence** | Often require pre-built playlists or manifests | **Fully standalone** — just paste a URL |
| **Processing chain** | Download video file only | Download → audio → transcription → **AI distillation** → archive |
| **Output** | Raw video file (GBs of storage) | Structured Markdown doc + cover image (+ optional video) |
| **Bilibili integration** | Basic yt-dlp support | **Deep Bilibili API integration** — metadata, dash streams, subtitles, error codes |
| **Searchability** | Filename only | Full-text searchable Markdown with TL;DR and timestamps |
| **Multi-episode series** | Manual processing | **Automatic parallel processing** across sub-agents with failure isolation |
| **Incremental updates** | Re-download everything | Detects existing archives, appends only new episodes |
| **Robustness** | Hardcoded, fragile | **28 engineering safeguards** — error mapping, JSON compatibility, URL retries, timeouts |

---

## ✨ Features

| Capability | Description |
|---|---|
| 🎬 **Multi-platform** | Bilibili, YouTube, and any yt-dlp supported platform (Douyin, Vimeo, Twitter, and more) |
| 🔊 **Audio extraction** | Downloads audio streams via Bilibili dash API (multi-codec) or yt-dlp |
| 📝 **Transcription** | Built-in subtitles first (Bilibili API + multi-format JSON parsing); falls back to OpenAI Whisper (base model) |
| 🌐 **Multi-language** | Auto-detects video language; supports Chinese, English, Japanese, Korean, and auto-detection mode |
| 🧠 **AI distillation** | Per-section: TL;DR, 3 core takeaways, key concepts, actionable techniques, gotchas, timestamped highlights |
| 📂 **Auto-classification** | Routes into 10 topic categories based on content analysis |
| 📄 **Structured archive** | Full Markdown docs: metadata card + embedded player + cover image + knowledge summary |
| 🔄 **Incremental updates** | Detects existing archives, extracts only new episodes/parts, appends to original doc |
| ⚡ **Parallel processing** | Multi-episode series split across sub-agents for concurrent transcription + distillation (3–4 episodes per batch) |
| 🛡️ **28 safeguards** | Error code mapping, subtitle JSON compatibility, fallback URL retries, disk space prechecks, 30-min timeout detection, PID isolation, and more |
| ⏱️ **Smart duration handling** | <60s: index only · 60s–3min: condensed · >3min: full comprehensive distillation |
| 🧹 **Self-cleaning** | Automatic temp file cleanup, error log preservation on failure, total processing time report |

---

## 🎯 Use Cases

- **Learning a new framework** — Archive a 20-episode tutorial series into a single searchable knowledge base. Find any concept by searching the Markdown file.
- **Conference talks & keynotes** — Capture the key insights from a 2-hour talk as a 5-minute read with timestamped highlights for revisiting specific moments.
- **Research & reference** — Build a personal video wiki: every technique, config snippet, or insight is timestamped and searchable.
- **Content creation** — Extract reference material, quotes, and techniques from creator videos for your own work.
- **Language learning** — Transcribe foreign-language videos (Chinese, English, Japanese, Korean); original text preserved with Chinese summaries appended.
- **Team knowledge sharing** — Share a distilled Markdown doc instead of a 2-hour video link — your team actually reads it.

---

## 🔧 Pipeline Overview

```
  Video URL
     │
     ▼
┌─────────────────┐
│ 0.  Dependency  │  Auto-install yt-dlp, ffmpeg, Whisper; preload ~140MB model; disk check (≥1GB)
│     Self-Check  │
└────────┬────────┘
         ▼
┌─────────────────┐
│ ½.  Network     │  Probe Bilibili API, YouTube, GitHub reachability before starting
│     Probe       │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 1.  Dedup       │  Scan existing archives → skip or enter incremental mode
└────────┬────────┘
         ▼
┌─────────────────┐
│ 2.  Platform    │  Bilibili → API (dash streams) │ YouTube/Other → yt-dlp
│     Routing     │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 3.  Metadata    │  Title, creator, cover, play count, date, duration, episode list
│                 │  (Full Bilibili error code mapping: -400/-403/-404/62002/62003/62004)
└────────┬────────┘
         ▼
┌─────────────────┐
│ 4.  Cover       │  Download + store alongside archive
│     Download    │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 5.  Language    │  Bilibili subtitle API / yt-dlp → Whisper model matching
│     Detection   │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 6.  Subtitle /  │  Built-in subtitles first (multi-format JSON) → Whisper fallback
│     Transcript  │  30-min timeout guard against stuck transcription
└────────┬────────┘
         ▼
┌─────────────────┐
│ 7.  AI          │  TL;DR + 3 takeaways + per-section: core ideas, techniques, gotchas, timestamps
│     Distill     │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 8.  Auto-       │  Route to 1 of 10 categories based on content + title analysis
│     Classify    │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 9.  Generate    │  Markdown doc: metadata card + embedded iframe + cover + full knowledge summary
│     Archive     │
└────────┬────────┘
         ▼
┌─────────────────┐
│ 10. Cleanup &   │  Total elapsed time, temp file removal, error log preservation (if any)
│     Report      │
└─────────────────┘
```

### 📊 Duration-Adaptive Processing

| Duration | Strategy |
|---|---|
| **< 60s** | Index only (metadata + cover), labeled "Short video, not distilled" |
| **60s – 3min** | Full pipeline, condensed extraction (1–2 key points per section) |
| **> 3min** | Full pipeline with comprehensive per-section distillation |

### 🔄 Incremental Updates

When a creator adds new episodes to an existing series, Chenmo:
1. Reads the existing archive to identify already-processed episodes
2. Compares against current episode list to find new ones
3. Processes only the new episodes (download → transcribe → distill)
4. Appends to the original document under an "📦 Incremental Update" section
5. Updates metadata (duration, play count, etc.)
6. If no new episodes → reports "Already fully archived"

---

## 📄 Sample Output

Here's what a generated knowledge document looks like:

```markdown
# Scrapling Cloudflare Bypass Deep Dive

> 🏷️ 尘墨·视频知识采集 | Archived: 2026-07-05 | Language: Chinese

![Cover](assets/尘墨-Scrapling-Cloudflare-Bypass-cover.jpg)

## 📋 Metadata

| Field | Value |
|---|---|
| Platform | Bilibili |
| Creator | Scrapling Official |
| URL | https://www.bilibili.com/video/BV1xx411c7mD |
| Published | 2026-06-15 |
| Duration | 18:42 |
| Views | 12,340 |
| Transcription | Built-in subtitles |

## 🎬 Video

<iframe src="https://player.bilibili.com/player.html?bvid=BV1xx411c7mD" ...></iframe>
🔗 [Watch on Bilibili](https://www.bilibili.com/video/BV1xx411c7mD)

## 📝 Knowledge Summary

### 📌 TL;DR
> **One-line summary:** StealthySession can bypass Cloudflare Turnstile reliably, but only in non-headless mode.
> **🔑 Three core takeaways:**
> 1. `solve_cloudflare=True` only works for Turnstile — hCaptcha needs a different approach
> 2. `headless=False` + `network_idle=True` is the critical combination for stability
> 3. Concurrent instances trigger IP rate-limiting — stagger or rotate proxies

---

## Section 3: Cloudflare Turnstile in Practice
> 🕐 Timestamp: 12:30 – 18:45

### 📌 Core Idea
StealthyFetcher's solve_cloudflare isn't a silver bullet — it needs proper browser fingerprinting and wait strategies.

### 🔑 Key Concepts
- `solve_cloudflare=True` targets Turnstile only; hCaptcha is unsupported
- `headless=False` is required for enhanced detection bypass
- Waiting for `network_idle` is more reliable than fixed `sleep 3s`

### 💡 Actionable Techniques
```python
with StealthySession(headless=False, solve_cloudflare=True) as s:
    page = s.fetch(url, network_idle=True, wait=2000)
```

### ⚠️ Gotchas
- Rapid requests to the same domain trigger IP rate-limiting — rotate proxies
- Never run multiple `solve_cloudflare` instances concurrently

### 🔖 Key Moments
- [14:02] Demo of Turnstile bypass in action
- [16:30] Author's mistake: `headless=True` causes infinite spinner
- [17:50] Final working solution revealed

---

*Generated by 尘墨·视频知识采集 Skill*
```

---

## 🧰 Tech Stack

| Component | Role |
|---|---|
| **yt-dlp** | Multi-platform video/audio download (Bilibili, YouTube, Douyin, Vimeo, Twitter, etc.) |
| **ffmpeg** | Audio format conversion (m4s → mp3, 128kbps) |
| **OpenAI Whisper (base)** | Speech-to-text fallback for videos without built-in subtitles; ~140MB model, multi-language |
| **Bilibili Player API** | Metadata, dash audio streams (multi-codec with fallback URLs), subtitle data |
| **OpenClaw sessions_spawn** | Parallel sub-agent scheduling for multi-episode series (3–4 episodes per batch) |

---

## 📂 Classification Categories

Chenmo automatically classifies videos into one of these categories:

| # | Category | Typical Content |
|---|---|---|
| 1 | AI Tools & Agents | Agent frameworks, OpenClaw, Cursor, AI programming tools |
| 2 | AI Image Generation | ComfyUI, Stable Diffusion, Midjourney, AI art |
| 3 | AI Video & Audio | Sora, Runway, AI video/audio generation |
| 4 | AI Writing & Creation | Prompt engineering, AI writing, ChatGPT techniques |
| 5 | Programming & Development | Pure coding, architecture, algorithms, backend/frontend |
| 6 | Product Design | UI/UX, Figma, product design |
| 7 | Reading & Philosophy | Book reviews, long-form essays, philosophy |
| 8 | Productivity Tools | Automation, AppleScript, keyboard shortcuts, efficiency |
| 9 | General Learning | Uncategorized educational content (labeled "pending classification") |
| 10 | Other | Falls back to root directory with review flag |

Classification priority: video title → transcript keywords → creator channel context.

---

## 🚫 Safety Boundaries

Chenmo explicitly **refuses** to process:

- 🚫 **Paywalled or members-only content** (even if technically downloadable)
- 🚫 **Copyright-protected material** with explicit "no download/redistribution" notices
- 🚫 **Private videos** involving personal privacy (unlisted personal videos of others)
- 🚫 **Adult or prohibited content**

When any of these are detected, Chenmo stops immediately and reports the reason.

---

## 🚀 Getting Started

### Prerequisites

```bash
# yt-dlp — multi-platform video download
pip install yt-dlp

# ffmpeg — audio/video conversion
brew install ffmpeg

# OpenAI Whisper — speech transcription (base model, ~140MB)
pip install openai-whisper
```

### Installation

```bash
# Install as an OpenClaw skill
npx skills add chenmo-video-kb
```

### Usage

Simply send a video URL to your OpenClaw agent — Chenmo activates automatically:

> *"帮我提炼这个视频 https://www.bilibili.com/video/BV1xx411c7mD"*

> *"Archive this tutorial series: https://www.youtube.com/watch?v=..."*

> *"下载这个B站视频并转成文字笔记"*

The agent detects the intent, runs the full pipeline, and reports back with the archive path and processing summary.

---

## ❓ FAQ

<details>
<summary><b>What happens if Whisper transcription fails?</b></summary>

Chenmo has a 30-minute timeout guard. If Whisper hangs or fails, the error is logged, and Chenmo reports the failure with the specific reason. You can retry or manually provide subtitles.
</details>

<details>
<summary><b>Can it handle videos in multiple languages?</b></summary>

Yes. Language auto-detection covers Chinese, English, Japanese, and Korean, with an auto-detect fallback. Foreign-language transcripts are preserved in the original language with a Chinese summary appended (not a full translation, to maintain accuracy).
</details>

<details>
<summary><b>How long does processing take?</b></summary>

Rough estimates for a 20-minute video:
- With built-in subtitles: ~2–4 minutes (metadata + AI distillation only)
- With Whisper transcription: ~8–15 minutes (audio download + transcription + distillation)

Multi-episode series benefit from parallel processing (3–4 episodes concurrently).
</details>

<details>
<summary><b>What platforms are supported?</b></summary>

Anything yt-dlp supports — Bilibili, YouTube, Douyin, Vimeo, Twitter/X, and hundreds more. Bilibili has additional deep API integration for metadata, subtitles, and audio streams.
</details>

<details>
<summary><b>Does it download the full video file?</b></summary>

By default, no — only the audio track is downloaded for transcription. The video itself is embedded via iframe player. Full video download is optional and only done when explicitly requested. This keeps storage minimal (a few KB of Markdown vs. GBs of video).
</details>

<details>
<summary><b>Where are archives stored?</b></summary>

All archives go to `knowledge/学习资源/<category>/` in your OpenClaw workspace, with cover images in an `assets/` subdirectory. The path is included in every completion report.
</details>

---

## 🤝 Contributing

This is an [OpenClaw skill](https://docs.openclaw.ai) — contributions, bug reports, and feature requests are welcome!

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

MIT © 2026

---

<p align="center">
  <sub>Built with ❤️ for the OpenClaw ecosystem · 尘墨出品</sub>
</p>
