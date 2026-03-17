# PixelVault Skill for Claude Code

Upload, manage, and serve images from your coding workflow — right inside Claude Code.

[PixelVault](https://pixelvault.dev) is agent-first image hosting for developers and AI coding agents. This skill lets Claude upload screenshots, diagrams, and generated images to PixelVault and get instant CDN URLs back.

## Skills

| Skill | Description |
|-------|-------------|
| `/pixelvault-upload <file>` | Upload image(s) and get CDN URLs |
| `/pixelvault-setup` | Install CLI and configure API key |
| `/pixelvault-list` | List recently uploaded images |

## Quick Start

1. Install the skill in Claude Code
2. Run `/pixelvault-setup` to configure your API key
3. Upload images with `/pixelvault-upload screenshot.png`

## How It Works

The skill wraps the [PixelVault CLI](https://www.npmjs.com/package/pixelvault-cli). When you (or Claude) upload an image:

1. Image is uploaded to PixelVault's edge infrastructure
2. Stored on Cloudflare R2 (zero egress fees)
3. Served globally via Cloudflare CDN
4. You get a permanent URL like `https://img.pixelvault.dev/proj_abc/img_xyz.png`

## Requirements

- Node.js 20+
- A PixelVault account (free tier: 100 images, 5 MB per file)

## Examples

```
> /pixelvault-upload screenshot.png
Uploaded: https://img.pixelvault.dev/proj_abc/img_xyz.png

> /pixelvault-upload *.png --folder diagrams
Uploaded: https://img.pixelvault.dev/proj_abc/img_001.png
Uploaded: https://img.pixelvault.dev/proj_abc/img_002.png

> /pixelvault-list --json
[{ "id": "img_xyz", "url": "https://...", "size": 45200, ... }]
```

## Links

- [PixelVault](https://pixelvault.dev) — Landing page
- [API Docs](https://pixelvault.dev/docs) — Full API reference
- [CLI](https://www.npmjs.com/package/pixelvault-cli) — CLI on npm
- [OpenAPI Spec](https://api.pixelvault.dev/openapi.json) — Machine-readable API spec

## License

MIT
