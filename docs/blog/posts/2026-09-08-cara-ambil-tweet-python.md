---
title: "Cara Mengambil Tweet Pakai Python — Setup, Auth, dan Fallback"
date: 2026-09-08
authors:
  - isra
categories:
  - Tooling
tags:
  - python
  - twitter
  - x
  - scraping
  - api
  - auth
  - pipeline
---

# Cara Mengambil Tweet Pakai Python — Setup, Auth, dan Fallback

Mengumpulkan tweet secara terstruktur bukan cuma soal panggil API — tapi juga siapin auth, handle rate limit, dan punya fallback kalau search ke-blokir. Inilangkah-langkahyang gw pake buat collect tweet komunitas tech Indonesia di techbro.

<!-- more -->

## Kenapa Bukan Library Siap-pake

Ada library seperti `twikit` yang janji ngambil tweet cuma 3 baris. Tapi di praktik, library itu sering:

- **Tergantung versi JS/XHR tertentu** — kalau Twitter update frontend, library mogok
- **Gabisa handle custom auth flow** — bearer token, CSRF cookie, authorize redirect, dll
- **Gabisa fallback graceful** — kalau search di-blokir, lib rapper cuma nemu exception, bukan fallback ke timeline

Solusi yang lebih robust: panggil Twitter API/HTTP endpoint langsung, ekstrak JSON, dan punya strategi fallback kalau satu endpoint mogok.

---

## Komponen Utama

Ada 3 layer yang biasa gw pakai:

### 1. Auth — Bearer + Cookie + CSRF

Endpoint Twitter X API butuh kombinasi:

- **Bearer Token** di `Authorization: Bearer ...`
- **Cookie** berisi `auth_token`, `ct0` (CSRF), `twid`, `kdt`
- **Header tambahan**: `X-Csrf-Token`, `X-Twitter-Active-User`, `X-Twitter-Auth-Type`
- **User-Agent** — bukan opsional, endpoint mati kalau tidak ada atau polanya mencurigakan

Contoh fungsi pembangun header:

```python
def _headers(creds):
    return {
        "Authorization": f"Bearer {creds['bearer']}",
        "X-Csrf-Token": creds["ct0"],
        "Cookie": (
            f'auth_token={creds["auth_token"]}; '
            f'ct0={creds["ct0"]}; '
            f'twid={creds.get("twid","")}; '
            f'kdt={creds.get("kdt","")}; '
            f'lang=id'
        ),
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                      "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36",
        "X-Twitter-Active-User": "yes",
        "X-Twitter-Auth-Type": "OAuth2Session",
    }
```

**Penting:** simpan creds di file (contoh: `creds.json`), jangan hardcode. File itu masuk `.gitignore`.

### 2. Collect — Search vs Timeline

Ada 2 endpoint utama:

**Search (endpoint `/api/2/search/adaptive.json`)**

Cocok kalau mau ambil tweet berdasarkan keyword. Responnya bentuk tree JSON dengan `globalObjects.tweets` + `globalObjects.users`.

```python
def raw_search(creds, q, count=20):
    url = (
        f"https://x.com/i/api/2/search/adaptive.json?"
        f"q={urllib.parse.quote(q)}"
        f"&result_filter=latest"
        f"&count={count}"
        f"&tweet_search_mode=live"
        f"&include_entities=true"
    )
    req = urllib.request.Request(url, headers=_headers(creds))
    with urllib.request.urlopen(req, timeout=25) as r:
        body = r.read().decode(errors="replace")
    if not body.strip():
        return []  # soft-block: body kosong
    d = json.loads(body)
    tw = d.get("globalObjects", {}).get("tweets", {})
    us = d.get("globalObjects", {}).get("users", {})
    out = []
    for tid, t in tw.items():
        u = us.get(t.get("user_id_str", ""), {})
        out.append({
            "id": tid,
            "text": t.get("text", ""),
            "created_at": t.get("created_at"),
            "user": u.get("screen_name", ""),
            "lang": t.get("lang", ""),
        })
    return out
```

**Problem:** endpoint ini sering **di-blokir secara soft** — HTTP 200, tapi body kosong, atau error di indeks parsing. Jadi fungsi search harus return `[]` kalau body kosong, bukan raise.

**Timeline (endpoint `/api/2/timeline/home.json`)**

Cocok buat fallback: ambil tweet yang muncul di home timeline account, lalu filter manual berdasarkan keyword.

```python
def raw_home_timeline(creds, count=40, cursor=None):
    url = (
        f'https://x.com/i/api/2/timeline/home.json?'
        f'count={count}'
        f'&include_entities=true'
        f'&latest=true'
    )
    if cursor:
        url += f'&cursor={urllib.parse.quote(cursor)}'
    req = urllib.request.Request(url, headers=_headers(creds))
    with urllib.request.urlopen(req, timeout=25) as r:
        return json.loads(r.read().decode())
```

Kalau mau ambil lebih dari 40 tweet, pake cursor pagination. Kuncinya ekstrak cursor dari `timeline.instructions`.

```python
cursor = None
for ins in d.get("timeline", {}).get("instructions", []):
    for e in (ins.get("addEntries", {}).get("entries", []) or
               ins.get("entries", [])):
        op = e.get("content", {}).get("operation", {})
        if op.get("cursor", {}).get("cursorType") == "Bottom":
            cursor = op["cursor"].get("value")
```

### 3. Fallback Strategy — Kalau Search Di-blokir

Flow yang gw pakai:

1. **Coba raw search per keyword** (max 20 tweet per keyword)
2. **Kalau search balikin `[]` / body kosong** → berarti di-blokir, lanjut ke timeline fallback
3. **Timeline fallback**: ambil beberapa halaman home timeline, filter tweet yang mengandung keyword
4. **Dedup** sebelum simpan — berdasarkan `tweet_id` dan `text_hash` (SHA256 dari teks lowercase)

Kenapa fallback perlu? Karena:

- Search bisa di-blokir per akun / per region / per keyword tertentu
- Timeline selalu bisa diakses kalau akunnya punya tweet (walaupun terbatas)
- Kombinasi keduanya luwih luas cakupan datanya

---

## Simpan & Dedup

Simpan di database (PostgreSQL / SQLite / CSV tergantung skala). Struktur minimal:

```sql
CREATE TABLE tweets (
    id TEXT PRIMARY KEY,
    text TEXT,
    created_at TIMESTAMP,
    username TEXT,
    lang TEXT,
    keyword TEXT,
    text_hash TEXT,
    collected_at TIMESTAMP DEFAULT NOW()
);
```

Dedup logic:

```python
def text_hash(t):
    return hashlib.sha256((t or "").strip().lower().encode()).hexdigest()[:16]

def save(conn, tweets, keyword):
    cur = conn.cursor()
    n = 0
    for t in tweets:
        h = text_hash(t["text"])
        cur.execute(
            "SELECT 1 FROM tweets WHERE text_hash=%s OR id=%s",
            (h, t["id"])
        )
        if cur.fetchone():
            continue
        cur.execute(
            "INSERT INTO tweets (id,text,created_at,username,lang,keyword,text_hash) "
            "VALUES (%s,%s,%s,%s,%s,%s,%s) ON CONFLICT DO NOTHING",
            (t["id"], t["text"], _parse_dt(t["created_at"]),
             t["user"], t["lang"], keyword, h)
        )
        n += 1
    conn.commit()
    cur.close()
    return n
```

Dua lapisan dedup:

- **id** — tweet ID unik, kalau ada di DB berarti udah pernah dikumpulkan
- **text_hash** — kalau tweet di-retweet atau repost dengan ID beda tapi teks sama, hash bikin deteksi duplikat

---

## Walkthrough Singkat

Alur script collect-nya kira-kira:

```python
import argparse, psycopg2

ap = argparse.ArgumentParser()
ap.add_argument("--max", type=int, default=200)
ap.add_argument("--keyword", type=str, default=None)
ap.add_argument("--timeline-only", action="store_true")
args = ap.parse_args()

creds = load_creds()
conn = psycopg2.connect(**PG)
ensure_schema(conn)

keywords = [args.keyword] if args.keyword else KEYWORDS
total = 0
used_search = False

for kw in keywords:
    if args.timeline_only:
        continue
    res = raw_search(creds, kw, count=20)
    if res:
        used_search = True
        saved = save(conn, res, kw)
        total += saved
        print(f"[search] {kw}: {len(res)} raw, {saved} baru")
    if total >= args.max:
        break
    time.sleep(3)

if not used_search and total == 0:
    print("[fallback] search di-blokir, pakai home timeline + filter keyword...")
    tl = collect_via_timeline(creds, keywords, pages=6)
    saved = save(conn, tl, "timeline-filter")
    total += saved
    print(f"[timeline] {len(tl)} match, {saved} baru")

print(f"\nTotal tweet terkumpul: {total}")
```

Fitur yang biasa gw tambahin:

- `--max` — batas tweet per run
- `--keyword` — fokus ke satu keyword
- `--timeline-only` — skip search, langsung timeline fallback
- Log per-keyword biar bisa monitor mana yang blocked

---

## Hal yang Sering Gagal (dan Solusi)

### 1. Body kosong (soft block)

Gejalanya: HTTP 200, tapi respons JSON kosong atau error parsing `globalObjects.

Solusi:

- Cek apakah cookie/expired — kalau `ct0` atau `auth_token` kadaluarsa, login ulang
- Cek apakah dia kena IP rate-limit — coba dari IP beda atau tunggu
- Kalau memang di-blokir, fall back ke timeline

### 2. Cursor pagination macet

Kalau cursornya null di tengah jalan, artinya timeline udah habis (daily tweet limit pequeño). Coba:

- Interval antar request lebih lama (misal `time.sleep(2-3)`)
- Kurangi `count` per request
- Gabung data dari beberapa hari

### 3. Token/cookie kadaluarsa

Twitter token (bearer, auth_token, ct0) punya masa aktif. Kalau script mulai balikin 0 tweet padahal dulu bisa, kemungkinan auth kedaluwarsa. Solusi:

- Refresh creds.json dari session browser terbaru
- Kalau pakai akun dedicated, pastikan session tetep hidup

### 4. Duplicate selama periode polling

Kalau collect setiap hari untuk keyword sama, tweet bakal banyak duplikat antar hari. Solusi:

- Pakai `text_hash` + `tweet_id` dedup
- Simpan juga `collected_at` biar bisa filtasi "tweet baru sejak collect terakhir"

---

## Setup Singkat

Apa yang perlu disiapin:

1. **Akun Twitter** dengan akses ke timeline yang diinginkan
2. **Cookie + bearer token** — bisa diambil dari DevTools Network (autentikasi X) atau session extension
3. **Python 3 + psycopg2** (kalau pakai PostgreSQL) atau `sqlite3` (kalau mau lebih simpel)
4. **Database** — PostgreSQL kalau mau query kompleks, SQLite kalau mau cepat setup
5. **Script collect** — contoh struktur seperti di atas

---

## Tips Praktis

- **Mulai dari 1-2 keyword dulu**, bukan 10 — biar gampang debug kalau ada endpoint yang mogok
- **Log per langkah** — berapa tweet raw, berapa yang masuk DB, berapa yang dedup
- **Test dengan timeline-only dulu** — kalau timeline jalan, berarti authnya ok, tinggal search-nya yang bermasalah
- **Simpan hasil di CSV/JSON juga** — selain database, biar ada backup plain-text yang bisa dibaca tanpa DB connection
- **Untuk dataset publik** — pastikan hanya tweet publik, tidak ada data pribadi di luar yang sudah tersedia di X

---

## Terkait

Beberapa tulisan lain yang mungkin relevan:

- [PySpark + Hudi + MinIO Local Development Setup](/blog/2025/09/15/pyspark-hudi-minio-local-development-setup/) — setup lingkungan data lokal
- [How I Use Claude as a Data Engineer](/blog/2026/04/16/how-i-use-claude-as-a-data-engineer/) — workflow data engineer sehari-hari
- [k3d nginx routes to node port, not NodePort](/til/) — catatan singkat infrastruktur

---

Jika ada pertanyaan, bisa kontak lewat GitHub atau email (lihat halaman About).
