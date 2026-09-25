---
draft: false
date: 2026-09-25
categories:
  - Tooling
slug: podcast-clips-auto-clipper
readtime: 5
---

# Podcast Clips — Automated Clipping Pipeline for Tempo Podcasts

I built an automated pipeline that monitors Tempo's podcast playlists (Jelasin Dong!, Bocor Alus Politik, Tukang Kupas Perkara), downloads new episodes, transcribes them, selects interesting moments via LLM, face-tracks and crops them to 9:16 with burned-in subtitles, and deploys the result to a web gallery — all without human intervention.

The result lives at **[clips.gcp.my.id](https://clips.gcp.my.id)** and the code is open source on **[github.com/israhabibi/podcast-clips](https://github.com/israhabibi/podcast-clips)**.

<!-- more -->

## The Pipeline

The entire flow is a chain of scripts, each responsible for one transformation:

1. **Monitor** — `monitor.py` checks RSS feeds of 3 Tempo YouTube playlists every 24 hours via cron (09:00 WIB).
2. **Download** — `yt-dlp` pulls the source video to `/tmp/podcast-clips/<episode-id>/`.
3. **Transcribe** — YouTube transcript API (lightweight) or faster-whisper (heavier but more accurate).
4. **Curate** — `curate.py` sends the transcript to a SumoPod LLM which picks 6–12 best moments.
5. **Cut & reframe** — `cut_smart.py` uses OpenCV face detection to track speakers, crops to vertical 9:16, and burns in subtitles with FFmpeg. This is the heaviest stage.
6. **Caption** — `caption.py` generates social captions and fetches related news links.
7. **Deploy** — clips are copied into Flask static files and the gallery refreshes.
8. **Upload** — optional YouTube Shorts & TikTok upload via OAuth API.

## Tech Stack

| Component | Tool |
|---|---|
| Web gallery | Flask + HTML/JS |
| Face detection | OpenCV (CPU, no GPU needed) |
| Video processing | FFmpeg |
| Transcription | faster-whisper / YouTube transcript API |
| Moment curation | SumoPod LLM (DeepSeek) |
| Hosting | GCP VM |
| Domain | clips.gcp.my.id (Cloudflare) |

## Why Build This?

Tempo produces hours of long-form political commentary every week. Most of it is too long to share as-is on platforms like YouTube Shorts, TikTok, or X. The pipeline turns a 45-minute podcast into a dozen 30–60 second clips automatically — each focused on one talking point, subtitled, and ready to publish.

Since the cron runs daily, new episode clips appear on the gallery without anyone touching the server.

## Try It

Visit **[clips.gcp.my.id](https://clips.gcp.my.id)** to browse the latest clips. The repo is at **[github.com/israhabibi/podcast-clips](https://github.com/israhabibi/podcast-clips)** — feel free to fork or adapt for your own content pipeline.
