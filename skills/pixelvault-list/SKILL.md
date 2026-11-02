---
name: pixelvault-list
description: List recently uploaded images on PixelVault with their CDN URLs. Use when the user asks to "see my uploads", "find that image URL", "what images have I hosted", "list my PixelVault images", or needs the link to a previously uploaded image.
args:
  - name: options
    description: Optional flags like --json for full metadata or --per-page 50 for more results
    required: false
---

# PixelVault List

List uploaded images and their CDN URLs.

## Prerequisites

The `pixelvault` CLI must be installed and configured. If not, tell the user to run `/pixelvault-setup`.

## Steps

1. **List images**:

```bash
# Default: 20 most recent, one URL per line
pixelvault list

# With full metadata
pixelvault list --json

# Pagination
pixelvault list --page 2 --per-page 50

# Only images tagged at upload with custom metadata (CLI 0.11.0+; all pairs must match)
pixelvault list --meta env=prod --meta run=42      # string values
pixelvault list --metadata '{"n":3}'               # typed values

# Search the AI descriptions (alt text, caption, OCR text) of images uploaded with --describe
# (CLI 0.12.0+; best match first; words are ANDed, "quoted phrases" and -word work)
pixelvault list --search "invoice 4821"
pixelvault list --search "login error" --meta env=staging --json
```

2. **Report** — Show the URLs (or metadata if `--json` was used) to the user.

## Output

- Default: one CDN URL per line to stdout
- `--json`: full response with id, url, mime_type, size, filename, folder, created_at, custom `metadata`, the AI `description` (when set), and pagination metadata
