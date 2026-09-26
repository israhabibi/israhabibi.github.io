---
title: "Automated Podcast Clipping Pipeline: Dari RSS ke Web Gallery"
date: 2026-09-25
authors:
  - isra
categories:
  - Tooling
tags:
  - python
  - flask
  - youtube
  - automation
  - pipeline
---

# Automated Podcast Clipping Pipeline: Dari RSS ke Web Gallery

Build automated pipeline yang download, clip, dan publish podcast ke YouTube Shorts, TikTok, dan web gallery — tanpa bayar API mahal.

<!-- more -->

## Motivation

Tempo podcast (Jelasin Dong!, Bocor Alus Politik, Tukang Kupas Perkara) publish episode baru 2-3x seminggu. Manual clipping: download → transcribe → pilih momen → crop 9:16 → burn subtitle → write caption → upload. Bisa 2-3 jam per episode.

Ditambah requirements:
- Free atau cheap API only (no $100/mo X API)
- Web gallery dengan ringkasan episode + klip
- Hook untuk X (Twitter) per episode
- YouTube Shorts + TikTok posting

## Pipeline Overview

```
YouTube RSS (monitor)
        ↓
yt-dlp (download)
        ↓
faster-whisper (transcribe, CPU)
        ↓
curate.py (LLM: summary + x_post + 6-12 clips)
        ↓
cut_smart.py (face-track 9:16 + ASS subtitles)
        ↓
caption.py (LLM: TikTok caption + news links)
        ↓
Deploy to Flask web + upload YouTube/TikTok
```

## Key Components

### 1. Face-Tracking 9:16 Crop (`cut_smart.py`)

AV1 codec YouTube tidak bisa di-decode OpenCV langsung. Sample frames via ffmpeg, baru deteksi wajah YuNet. Crop position di-interpolasi smooth (median filter → CubicSpline → speed clamp 250 px/s).

```python
# Face tracking: 24 samples, smooth crop
pts, w, h = face_positions(EP, start, dur)
crop_w = int(h * 9 / 16)
track = smooth_track(pts, dur, crop_w, w)
```

### 2. One-LLM-Call Curation (`curate.py`)

Satu prompt ke SumoPod dapat:
- **episode_summary** — 3-5 kalimat untuk web
- **x_post** — hook + hashtag untuk X (link ke web gallery)
- **6-12 clip moments** — timestamp, title, hook

```bash
PODCAST_WORK_DIR=/tmp/podcast-clips/episode-id \
  python curate.py MiniMax-M2.7-highspeed jelasin-dong "Judul Episode"
```

Output: `episode_data.json`

### 3. Caption dengan Force-Appended Links (`caption.py`)

LLM write caption tanpa link, lalu programmatically append 3 news URLs hasil web search. Ini bypass LLM yang suka hallucinate ataupotek link.

### 4. Web Gallery (Flask)

Episode landing card (summary + x_post preview) → scroll Shorts-style 100dvh snap → progress dots → mute toggle. Serving via `/static/clips/<podcast>/<episode>/`.

```
https://clips.gcp.my.id/clips/jelasin-dong/2026-09-25_judul-episode
```

## Architecture

```
Cloudflare Tunnel (clips.gcp.my.id)
        ↓
Flask (port 5000, ProxyFix x_proto)
        ↓
/auth → YouTube OAuth (PKCE, state+verifier in session)
/oauth → token callback
/static/clips/ → video files
```

OAuth pitfalls yang pernah kena:
- Google require PKCE, code_verifier harus persist di session
- Cloudflare tunnel → Flask lihat HTTP, perlu ProxyFix
- Channel picker → salah channel = metadata update fail

## Cost

| Komponen | Biaya |
|---|---|
| Transcription | CPU (faster-whisper small) |
| LLM (curate + caption) | SumoPod ~$0.002/episode |
| YouTube API | Free |
| TikTok API | Free |
| Hosting | VPS + Cloudflare Tunnel |
| X posting | Manual (text generated, no API) |

## Future Work

- Auto-post ke X (butuh $100/mo X API — skipped)
- Batch upload scheduling
- A/B testing caption variants
- Multi-language subtitle generation

## Source Code

Pipeline ada di [github.com/israhabibi/podcast-clips](https://github.com/israhabibi/podcast-clips). Skill Hermes di `hermes-skill/`.

## Lessons Learned

1. **AV1 decode**: OpenCV can't decode AV1 → use ffmpeg for frames first
2. **ASS header**: `[V4 Styles]` parses garbage → must be `[V4+ Styles]` with full Format line
3. **Crop smoothing**: linear interp = jittery crop → median filter + cubic spline + speed clamp
4. **Caption links**: don't trust LLM to include URLs → force-append programmatically
5. **OAuth state**: PKCE requires code_verifier in session, not just state
