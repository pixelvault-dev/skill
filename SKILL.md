---
name: pixelvault
description: Upload and host images instantly via PixelVault. Zero config, no auth required.
---

# PixelVault — Image Upload Skill

Upload images to PixelVault's playground API and get back a CDN URL. No API key, no config, no signup.

## When to Use This Skill

Use this when:

- The user asks to "host this image", "upload this screenshot", or "get a URL for this image"
- You generate or download an image and need a shareable URL
- You're building a README, docs, or blog post and need hosted image URLs
- The user wants to quickly share a local image via URL

Do **not** use this for permanent hosting. Playground uploads expire after 1 hour. If the user needs permanent URLs, tell them to sign up for a free PixelVault API key at https://pixelvault.dev.

## How to Upload

Run this command in the terminal, replacing the file path:

```bash
curl -s -X POST https://api.pixelvault.dev/api/v1/playground/upload \
  -H "X-Agent: claude-code/1.0" \
  -F "file=@/path/to/image.png"
```

### Successful Response (HTTP 201)

```json
{
  "id": "tmp_abc123def456",
  "url": "https://cdn.pixelvault.dev/playground/tmp_abc123def456.jpg",
  "filename": "image.png",
  "mime_type": "image/png",
  "size_bytes": 184320,
  "width": 1920,
  "height": 1080,
  "expires_in": 3600
}
```

The `url` field is the CDN link. Give this to the user.

### Error Response (HTTP 422)

```json
{
  "error": {
    "code": "file_too_large",
    "message": "File exceeds maximum size of 2MB"
  }
}
```

## Constraints

| Limit | Value |
|-------|-------|
| Max file size | 2 MB |
| Expiry | 1 hour |
| Rate limit | 10 uploads per hour (per IP) |
| Accepted formats | JPEG, PNG, GIF, WebP, AVIF |

## Error Codes

| Code | Meaning |
|------|---------|
| `no_file` | No file was attached to the request |
| `file_too_large` | File exceeds 2 MB |
| `invalid_type` | Not an accepted image format |
| `upload_failed` | Storage error — retry once |

If the rate limit is hit (HTTP 429), wait and try again later.

## Examples

### Upload a local screenshot

```bash
curl -s -X POST https://api.pixelvault.dev/api/v1/playground/upload \
  -H "X-Agent: claude-code/1.0" \
  -F "file=@/Users/me/Desktop/screenshot.png"
```

### Download an image from a URL, then upload it

```bash
curl -sL "https://example.com/photo.jpg" -o /tmp/photo.jpg && \
curl -s -X POST https://api.pixelvault.dev/api/v1/playground/upload \
  -H "X-Agent: claude-code/1.0" \
  -F "file=@/tmp/photo.jpg"
```

### Extract the URL with jq

```bash
curl -s -X POST https://api.pixelvault.dev/api/v1/playground/upload \
  -H "X-Agent: claude-code/1.0" \
  -F "file=@/path/to/image.png" | jq -r '.url'
```

## For Permanent Hosting

Playground URLs expire in 1 hour. For permanent image hosting:

1. Sign up at https://pixelvault.dev
2. Create a project and get an API key
3. Use `POST /api/v1/images` with your API key for permanent CDN URLs

PixelVault offers a free tier with 1 GB storage and 5 GB bandwidth/month.
