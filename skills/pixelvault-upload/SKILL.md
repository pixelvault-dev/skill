---
name: pixelvault-upload
description: Upload image files to PixelVault and get a public CDN URL back. Use when the user says "screenshot this and host it", "upload this image", "host this mockup", "give me a URL for this diagram", "attach this screenshot to the PR/issue", or when you generate, render, or save an image (screenshot, diagram, chart, build artifact) that needs a shareable link. Ideal for embedding images in GitHub PRs, issues, and comments.
args:
  - name: files
    description: File path(s) or glob pattern to upload (e.g. screenshot.png, ./images/*.jpg)
    required: true
  - name: folder
    description: Optional folder to organize images in
    required: false
---

# PixelVault Upload

Upload image files to PixelVault and return CDN URLs.

## Prerequisites

The `pixelvault` CLI must be installed and configured. If the command is not found, install it:

```bash
npm install -g pixelvault-cli
```

If no API key is configured, tell the user to run `/pixelvault-setup` first or set `PIXELVAULT_API_KEY` in their environment.

## Steps

1. **Check CLI availability** — Run `which pixelvault` to confirm the CLI is installed.

2. **Check auth** — Run `pixelvault whoami --json` to verify an API key is configured. If it fails, stop and tell the user to run `/pixelvault-setup` or set `PIXELVAULT_API_KEY`.

3. **Upload** — Run the upload command:

```bash
# Single file
pixelvault upload $ARGUMENTS

# With folder
pixelvault upload <file> --folder <folder>

# Multiple files (shell glob)
pixelvault upload *.png --folder screenshots

# Private — returns a signed URL only holders of the link can open
pixelvault upload mockup.png --private
pixelvault upload mockup.png --private --expires 7d
```

4. **Report** — The CLI prints one CDN URL per line to stdout. Report these URLs back to the user. A public URL is permanent and globally cached; a `--private` URL is a signed link that expires (default 7 days).

## Output Contract

- The CLI prints **only URLs** to stdout (one per line)
- Human-readable messages go to stderr
- Use `--json` if you need metadata (id, mime_type, size, filename)

## Error Handling

| Error | Action |
|-------|--------|
| `command not found: pixelvault` | Tell user to `npm install -g pixelvault-cli` |
| `No API key configured` | Tell user to run `/pixelvault-setup` or set `PIXELVAULT_API_KEY` |
| `401 Unauthorized` | API key is invalid — tell user to run `pixelvault login` |
| `402 ... private images` | Free plan's private-image limit reached — tell user to delete some or upgrade |
| `413 Payload Too Large` | File exceeds plan limit (5 MB free, 50 MB paid) |
| `415 Unsupported Media Type` | File is not a supported image format |

## Examples

```bash
# Upload a screenshot, get the URL
pixelvault upload screenshot.png
# → https://img.pixelvault.dev/proj_abc/img_xyz.png

# Upload with JSON metadata
pixelvault upload diagram.svg --json
# → { "id": "img_xyz", "url": "https://...", "size": 12400, ... }

# Bulk upload
pixelvault upload ./assets/*.png --folder assets

# Private mockup, link valid for 24 hours
pixelvault upload mockup.png --private --expires 24h
# → https://img.pixelvault.dev/proj_abc/cp/i/img_xyz.png?token=…&expires=…
```

## Private uploads

Use `--private` when the image shouldn't be publicly guessable — an internal
mockup, a CI screenshot, anything you only want to share via the link itself.
The returned URL is a **signed** URL: it works for anyone who has the full link
and stops working when it expires.

- `--private` alone → default lifetime (7 days).
- `--private --expires <duration>` → custom lifetime. Accepts `30m`, `12h`,
  `7d`, or a number of seconds. Min 60s, max 30d.
- Requires a **secret** API key (the default from `pixelvault login` / setup);
  publishable keys can't create private uploads.
- `--expires` only applies together with `--private`.

## Transform the result

Every returned URL supports on-the-fly transforms — just append query params (no
re-upload): `?w=400` (thumbnail), `?fmt=webp`, `?segment=foreground` (transparent
cut-out), `?saturation=0` (grayscale), `?tile=logo.png` (watermark). For the full
vocabulary use `/pixelvault-transform`.
