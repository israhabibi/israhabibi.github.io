---
draft: false
date: 2026-09-25
categories:
  - Tooling
slug: podcast-clips-auto-clipper
readtime: 5
---

# Podcast Clips — Pipeline Otomatis Potong Klip Podcast Pake AI

Pernah nonton podcast panjang 45 menit tapi cuma butuh 30 detik bagian yang menarik? Saya bikin pipeline yang **ngerjain itu otomatis** buat 3 podcast Tempo — ngambil momen menarik, potong 9:16, kasih subtitle, dan deploy ke web. Semua dari monitor RSS, tanpa disentuh tangan.

Live di **[clips.gcp.my.id](https://clips.gcp.my.id)** — repo: **[github.com/israhabibi/podcast-clips](https://github.com/israhabibi/podcast-clips)**

<!-- more -->

## Isi Podcast yang Diproses

Ada 3 playlist Tempo yang di-monitor tiap hari:

- **Jelasin Dong!** — podcast politik yang ngejelasin isu terkini
- **Bocor Alus Politik** — politik daleman, gaya santai
- **Tukang Kupas Perkara** — bedah kasus hukum

Hasil klip bisa dilihat langsung di **[clips.gcp.my.id](https://clips.gcp.my.id)** dalam bentuk YouTube Shorts–style gallery.

## Pipeline Lengkap

### Yang Pake LLM (AI):
1. **Kurasi Momen** (`curate.py`) — transcript dikirim ke SumoPod LLM. Model milih 6—12 momen terbaik dari 45 menit podcast.
2. **Buat Caption** (`caption.py`) — LLM nulis caption pendek buat tiap klip plus nyari berita terkait biar konteksnya nyambung.

### Yang Nggak Pake LLM (CPU lokal doang):
1. **Monitor RSS** (`monitor.py`) — tiap jam 09:00 WIB cek playlist YouTube, kalau ada episode baru langsung download.
2. **Download** — `yt-dlp` ambil video mentah dari YouTube.
3. **Transkrip** — YouTube transcript API (ringan) atau faster-whisper (lebih akurat, tapi berat di CPU).
4. **Potong & Reframe** (`cut_smart.py`) — ini bagian paling berat: OpenCV deteksi wajah, crop ke vertikal 9:16, burn-in subtitle pake FFmpeg.
5. **Upload** — kirim ke YouTube Shorts & TikTok via API OAuth.

### Deploy:
1. Klip yang udah dipotong di-copy ke Flask static folder.
2. Web gallery (`clips.gcp.my.id`) otomatis refresh.
3. Hosting di **GCP VM** — domain pake Cloudflare.
4. **Cron daily** 09:00 WIB jalan di VPS yang sama — pipeline penuh tanpa intervensi.

## Arsitektur

```bash
podcast-clips/
├── app/               # Flask web app (clips.gcp.my.id)
├── scripts/           # Upload ke YouTube/TikTok
├── curate.py          # LLM pilih momen
├── cut_smart.py       # Face-track + crop + subtitle
├── caption.py         # LLM generate caption
└── monitor.py         # RSS monitor 3 playlist
```

Stack lengkap: **Flask + OpenCV + FFmpeg + SumoPod LLM + GCP VM**.

## Catatan Penting

- **Transkrip:** faster-whisper makan CPU gede. Kalau lagi buru-buru, fallback ke YouTube transcript API — gratis dan ringan.
- **Cut & reframe:** tahap paling berat. 45 menit video butuh ~10-15 menit proses di VPS.
- **Repo cuma berisi kode.** Video hasil pipeline nggak masuk Git — cuma static folder di server yang nyimpen.

---

**Part 2** bakal gw tulis: cara bikin OAuth auth untuk YouTube/TikTok upload, cara setting auto-post, dan detail teknikal tiap komponen.
