<p align="center">
  <a href="https://deploy.workers.cloudflare.com/?url=https://github.com/your-username/facebook-scraper-api">
    <img src="https://deploy.workers.cloudflare.com/button" alt="Deploy to Cloudflare Workers" />
  </a>
</p>

<h1 align="center">📘 Facebook Video & Reels Scraper API</h1>

<p align="center">
  <strong>High-performance, serverless Facebook Video, Reels, Watch, and Share link scraper and downloader API designed for Cloudflare Workers.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-Cloudflare_Workers-F38020?style=for-the-badge&logo=cloudflare&logoColor=white" alt="Cloudflare Workers" />
  <img src="https://img.shields.io/badge/Runtime-V8_Worker_Isolate-3178C6?style=for-the-badge&logo=javascript&logoColor=white" alt="V8 Engine" />
  <img src="https://img.shields.io/badge/Max_Download-650_MB_Cap-E50914?style=for-the-badge" alt="650 MB Limit" />
  <img src="https://img.shields.io/badge/CORS-Universal_(*)-success?style=for-the-badge" alt="CORS" />
  <img src="https://img.shields.io/badge/Dependencies-Zero_npm-black?style=for-the-badge" alt="Zero Dependencies" />
</p>

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Key Features](#-key-features)
- [Architecture & 3-Engine Waterfall](#-architecture--3-engine-waterfall)
- [Supported Facebook Link Formats](#-supported-facebook-link-formats)
- [650 MB Download Limit & Rate Protection](#-650-mb-download-limit--rate-protection)
- [Raw File Size Reporting](#-raw-file-size-reporting)
- [API Endpoints Reference](#-api-endpoints-reference)
  - [1. Service Health & Status](#1-service-health--status)
  - [2. Scrape Facebook Video & Reels Metadata](#2-scrape-facebook-video--reels-metadata)
  - [3. Direct Download with Custom Filename](#3-direct-download-with-custom-filename)
  - [4. Dynamic Media Stream Proxy (Range Seeking)](#4-dynamic-media-stream-proxy-range-seeking)
- [JSON Response Schema](#-json-response-schema)
- [Deployment Guide](#-deployment-guide)
  - [Option 1: One-Click Deploy Button](#option-1-one-click-deploy-button)
  - [Option 2: Cloudflare Dashboard (Web UI)](#option-2-cloudflare-dashboard-web-ui)
  - [Option 3: Wrangler CLI](#option-3-wrangler-cli)
- [Client Integration Examples](#-client-integration-examples)
  - [cURL](#curl)
  - [JavaScript / TypeScript (Browser & Node.js)](#javascript--typescript)
  - [Python](#python)
- [Configuration (`wrangler.toml`)](#-configuration-wranglertoml)
- [License & Disclaimer](#-license--disclaimer)

---

## 🌟 Overview

The **Facebook Video & Reels Scraper API** is an edge-native REST API built for Cloudflare Workers. It extracts high-definition (HD 1080p/720p) and standard-definition (SD 360p/480p) MP4 video streams, exact file sizes in raw bytes and formatted strings, video duration, and thumbnails from public Facebook videos, Reels, and Watch posts.

---

## ✨ Key Features

- **3 Independent Third-Party Scraping Engines:** Parallel multi-engine architecture querying Facebook Embed Plugin, Prexzy AIO, and Cobalt Edge with automatic failover.
- **HD & SD MP4 Stream Extraction:** Automatically resolves both `HD` (High Definition) and `SD` (Standard Definition) progressive streams directly from Facebook's global CDN (`*.fbcdn.net`).
- **Raw File Size Data:** Returns both raw integer bytes (`size`) and human-readable formatted sizes (`size_formatted`) for every available quality.
- **650 MB Download Protection:** Enforces a strict `650 MB` limit on proxy downloads to prevent Worker memory exhaustion and timeouts.
- **Shortlink & Share Link Expansion:** Automatically resolves mobile app redirects (`fb.watch/...`, `facebook.com/share/v/...`, `facebook.com/share/r/...`).
- **Direct Edge Streaming (`/api/stream`):** Implements `Accept-Ranges: bytes` and `206 Partial Content` for HTML5 video player embedding and seeking.
- **Direct Download Proxy (`/api/download`):** Sets `Content-Disposition: attachment` with sanitized video title filenames, bypassing Facebook CDN hotlink restrictions.
- **Universal CORS:** Standard CORS headers enabled on all responses for seamless client-side frontend and bot integration.
- **Zero Runtime Dependencies:** Requires zero external npm packages and runs purely within the standard V8 Worker isolate.

---

## 🏗 Architecture & 3-Engine Waterfall

```text
                                Client Request
                                      │
         ┌────────────────────────────┴────────────────────────────┐
         │                                                         │
    /api/scrape                                           /api/download / /api/stream
  (Extract Metadata)                                    (Dynamic Streaming & Proxy)
         │                                                         │
         ▼                                                         ▼
  Shortlink Resolver:                                     Content-Length Inspection:
  Expands fb.watch / share links                          ├── If Size > 650 MB -> HTTP 413
         │                                                └── If Size <= 650 MB -> Stream directly
         ▼                                                         │
  Multi-Engine Parallel Query:                                     ▼
  ├── Engine 1: facebook.com/plugins/video.php            - Bypasses CDN 403 Forbidden
  ├── Engine 2: prexzyapis.com/download/aio               - Content-Disposition: attachment
  ├── Engine 3: cobaltapi.cjs.nz                          - Supports 206 Partial Content (Seeking)
  └── Fallback: facebook.com (Direct Web SSR)
```

| Engine | Source / Host | Purpose | Status |
|:---|:---|:---|:---|
| **Engine 1** | `facebook.com/plugins/video.php` | Native Facebook embed plugin providing direct HD & SD streams | Primary |
| **Engine 2** | `prexzyapis.com/download/aio` | Multi-quality stream extractor with duration and caption metadata | Primary |
| **Engine 3** | `cobaltapi.cjs.nz` | Unrestricted mirror engine returning direct `fbcdn.net` video streams | Primary |
| **Fallback** | `facebook.com` | Direct web page SSR hydration parser | Standby |

---

## 🔗 Supported Facebook Link Formats

- **Facebook Watch:** `https://www.facebook.com/watch/?v=10153231379946729`
- **Facebook Reels:** `https://www.facebook.com/reel/1535656380759655/`
- **Facebook Share Links:** `https://www.facebook.com/share/v/1DPWguWCnA/` or `https://www.facebook.com/share/r/1DPWguWCnA/`
- **Mobile Shortlinks:** `https://fb.watch/example123/`
- **User Video Posts:** `https://www.facebook.com/username/videos/10153231379946729/`
- **Numeric Video IDs:** `10153231379946729`

---

## 🛡 650 MB Download Limit & Rate Protection

To guarantee stable streaming, protect Worker memory, and adhere to Cloudflare Worker limits, the API enforces a strict **650 MB (`681,574,400` bytes)** maximum download cap:

1. **Header-Level Rejection:** If upstream `Content-Length` exceeds `681,574,400` bytes, the Worker immediately terminates the connection and returns **HTTP 413 (Payload Too Large)** without consuming Worker bandwidth.
2. **Chunk-Level Stream Guard:** For streams without a `Content-Length` header or chunked transfers, an internal `TransformStream` counts bytes transmitted in real-time. If the stream reaches 650 MB, it safely cuts the connection.
3. **Scrape Response Flagging:** Every video item in `/api/scrape` includes `download_allowed: true/false`, indicating whether the format fits within the 650 MB limit.

### HTTP 413 Error Response Example:
```json
{
  "status": "error",
  "code": 413,
  "message": "File size exceeds the 650 MB download limit",
  "file_size": 734003200,
  "file_size_formatted": "700 MB",
  "max_download_size": 681574400,
  "max_download_size_formatted": "650 MB"
}
```

---

## 📊 Raw File Size Reporting

Every media format exposes its exact byte count alongside its human-readable representation:

- `size`: Unsigned integer representing exact bytes (e.g. `5388007`)
- `size_formatted`: Formatted string (e.g. `"5.14 MB"`, `"1.13 MB"`)
- `download_allowed`: Boolean indicating whether file size is `<= 650 MB`

---

## 📡 API Endpoints Reference

Base URL: `https://<your-worker-name>.<your-subdomain>.workers.dev`

---

### 1. Service Health & Status

Returns server status, engine statuses, and download limit configurations.

- **Method:** `GET`
- **Path:** `/` or `/api` or `/health`

```bash
curl "https://your-worker.workers.dev/health"
```

```json
{
  "status": "ok",
  "timestamp": "2026-10-26T12:00:00.000Z"
}
```

---

### 2. Scrape Facebook Video & Reels Metadata

Extracts complete video metadata, stream links, and size information across all available resolutions.

- **Method:** `GET` or `POST`
- **Path:** `/api/scrape` *(Aliases: `/scrape`, `/api/video`, `/api/info`)*

#### Query Parameters:

| Parameter | Type | Required | Description |
|:---|:---|:---|:---|
| `url` / `id` | string | Yes | Facebook video / reels / watch / share URL, or numeric video ID |

#### Example Request:

```bash
curl "https://your-worker.workers.dev/api/scrape?url=https://www.facebook.com/watch/?v=10153231379946729"
```

#### JSON Response Schema:

```json
{
  "status": "success",
  "code": 200,
  "data": {
    "engine": "engine-1 (facebook-plugin)",
    "id": "10153231379946729",
    "title": "How to share with just friends",
    "duration": 74,
    "thumbnail": "https://graph.facebook.com/10153231379946729/picture?type=large",
    "max_download_size": 681574400,
    "max_download_size_formatted": "650 MB",
    "engines": {
      "active": "engine-1",
      "engine_1": "facebook.com/plugins/video.php",
      "engine_2": "prexzyapis.com",
      "engine_3": "cobaltapi.cjs.nz",
      "fallback": "facebook.com"
    },
    "sources": {
      "facebook_plugin": {
        "status": "active",
        "host": "facebook.com/plugins/video.php",
        "hd_url": "https://video-sea5-1.xx.fbcdn.net/...",
        "sd_url": "https://video-sea5-1.xx.fbcdn.net/...",
        "duration": 74
      },
      "prexzy_aio": {
        "status": "active",
        "host": "prexzyapis.com",
        "title": "How to share with just friends",
        "duration": 74,
        "hd_url": "https://video.fsgn5-21.fna.fbcdn.net/...",
        "sd_url": "https://video.fsgn5-9.fna.fbcdn.net/..."
      },
      "cobalt_edge": {
        "status": "active",
        "host": "cobaltapi.cjs.nz",
        "url": "https://video.fsyd16-2.fna.fbcdn.net/...",
        "filename": "facebook_10153231379946729.mp4"
      }
    },
    "media": {
      "video": {
        "quality": "HD",
        "size": 5388007,
        "size_formatted": "5.14 MB",
        "download_allowed": true,
        "url": "https://video-sea5-1.xx.fbcdn.net/...",
        "qualities": [
          {
            "quality": "HD",
            "resolution": "720p / 1080p",
            "size": 5388007,
            "size_formatted": "5.14 MB",
            "download_allowed": true,
            "url": "https://video-sea5-1.xx.fbcdn.net/...",
            "download_url": "https://your-worker.workers.dev/api/download?url=...",
            "stream_url": "https://your-worker.workers.dev/api/stream?url=..."
          },
          {
            "quality": "SD",
            "resolution": "360p / 480p",
            "size": 1180614,
            "size_formatted": "1.13 MB",
            "download_allowed": true,
            "url": "https://video-sea5-1.xx.fbcdn.net/...",
            "download_url": "https://your-worker.workers.dev/api/download?url=...",
            "stream_url": "https://your-worker.workers.dev/api/stream?url=..."
          }
        ],
        "download_url": "https://your-worker.workers.dev/api/download?url=...",
        "stream_url": "https://your-worker.workers.dev/api/stream?url=..."
      }
    }
  }
}
```

---

### 3. Direct Download with Custom Filename

Triggers direct browser download (`Content-Disposition: attachment`). Resolves the stream dynamically and enforces the 650 MB limit.

- **Method:** `GET`
- **Path:** `/api/download` *(Alias: `/download`)*

#### Query Parameters:

| Parameter | Type | Required | Description |
|:---|:---|:---|:---|
| `url` / `id` | string | Yes | Direct stream URL or Facebook post URL / video ID |
| `quality` | string | No | Preferred resolution: `HD` or `SD` (Default: `HD`) |
| `filename` | string | No | Custom file name (e.g. `MyVideo.mp4`) |

#### Examples:

```bash
# Download HD MP4 Video
curl -L -O "https://your-worker.workers.dev/api/download?id=10153231379946729&quality=HD"

# Download SD MP4 Video
curl -L -O "https://your-worker.workers.dev/api/download?id=10153231379946729&quality=SD"
```

---

### 4. Dynamic Media Stream Proxy (Range Seeking)

Streams video with `Content-Disposition: inline` and full `Accept-Ranges: bytes` support. Ideal for HTML5 `<video>` player embedding, mobile apps, and seeking.

- **Method:** `GET`
- **Path:** `/api/stream` *(Alias: `/stream`)*

```html
<!-- HTML5 Video Embedding with Seeking -->
<video controls width="100%" height="auto" src="https://your-worker.workers.dev/api/stream?id=10153231379946729&quality=HD"></video>
```

---

## 🚀 Deployment Guide

### Option 1: One-Click Deploy Button

Click the banner below to fork and deploy directly to your Cloudflare account in seconds:

[![Deploy to Cloudflare Workers](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/your-username/facebook-scraper-api)

---

### Option 2: Cloudflare Dashboard (Web UI)

1. Log into your [Cloudflare Dashboard](https://dash.cloudflare.com/).
2. Navigate to **Compute (Workers) -> Workers & Pages**.
3. Click **Create Application** -> **Create Worker**.
4. Choose a name (e.g., `facebook-scraper-api`) and click **Deploy**.
5. Once created, click **Edit code** in the top-right corner.
6. Paste the entire contents of [`src/index.js`](./src/index.js) into the code editor.
7. Click **Deploy**. Your API is live worldwide instantly!

---

### Option 3: Wrangler CLI

Ensure you have [Node.js](https://nodejs.org/) installed, then run:

```bash
# Clone the repository
git clone https://github.com/your-username/facebook-scraper-api.git
cd facebook-scraper-api

# Install Wrangler globally or locally
npm install -g wrangler

# Authenticate with Cloudflare
wrangler login

# Deploy directly to Cloudflare Workers
wrangler deploy
```

---

## 💻 Client Integration Examples

### cURL

```bash
# Scrape Facebook video
curl -s "https://your-worker.workers.dev/api/scrape?url=https://www.facebook.com/watch/?v=10153231379946729" | jq .
```

### JavaScript / TypeScript

```javascript
async function getFacebookVideo(fbUrl) {
  const endpoint = `https://your-worker.workers.dev/api/scrape?url=${encodeURIComponent(fbUrl)}`;
  const res = await fetch(endpoint);
  const data = await res.json();
  
  if (data.status === "success") {
    console.log("Title:", data.data.title);
    console.log("HD Video Download:", data.data.media.video.download_url);
    console.log("File Size:", data.data.media.video.size_formatted);
  }
}
```

### Python

```python
import requests

def scrape_facebook(url: str):
    api_url = "https://your-worker.workers.dev/api/scrape"
    response = requests.get(api_url, params={"url": url})
    data = response.json()
    
    if data.get("status") == "success":
        video = data["data"]["media"]["video"]
        print(f"Title: {data['data']['title']}")
        print(f"HD Download: {video['download_url']}")
        print(f"Size: {video['size_formatted']}")
        return data["data"]
    return None
```

---

## ⚙ Configuration (`wrangler.toml`)

Your `wrangler.toml` file should look like this:

```toml
name = "facebook-scraper-api"
main = "src/index.js"
compatibility_date = "2024-03-20"
```

---

## 📜 License & Disclaimer

This project is licensed under the MIT License.

*Disclaimer: This project is intended for educational, testing, and personal backup purposes. Please respect Facebook's Terms of Service and content creators' intellectual property rights.*
