---
tags:
  - image hosting
  - image hosting for agents
  - video hosting
  - CDN
  - screenshots
  - image management
  - diagrams
tools:
  - PixelVault
  - PixelVault CLI
  - PixelVault MCP
  - Claude Code
  - Codex
  - Cursor
  - Cloudflare R2
  - Cloudflare CDN
companies:
  - Cloudflare
dependencies:
  - Node.js
---
# PixelVault — Media Hosting for AI Agents

Agent skills for hosting images and MP4 videos: upload a file and get a permanent
CDN URL back from Claude Code, Codex, Cursor, or **any agent that reads
`SKILL.md`**.

[PixelVault](https://pixelvault.dev) is agent-first media hosting for developers
and AI coding agents. These skills let an agent upload screenshots, diagrams,
mockups, generated images, and MP4 videos and get an instant, globally cached
CDN URL without a manual dashboard.

**When it triggers:** "screenshot this and host it", "upload this image and give
me the URL", "host this MP4", "give me a URL for this video", or any task that
produces media needing a shareable link.

## Why PixelVault for agents

- **Just an API key** — no crypto wallet, no per-upload payment. A permanent free tier, not a trial.
- **Permanent, global URLs** — served from Cloudflare's edge CDN with zero egress fees.
- **Built for discovery** — ships a Claude Code skill, a remote **MCP server**, `llms.txt`, and an OpenAPI 3.1 spec so any agent can find and call it.
- **Works with any agent** — Claude Code, Codex, Cursor, Gemini CLI, or paste `SKILL.md` into custom instructions.

## Skills

| Skill | Description |
|-------|-------------|
| `/pixelvault-upload <file>` | Upload images or MP4 videos and get CDN URLs |
| `/pixelvault-host-image` | Zero-install: host a file or URL with one `curl`, no signup (keyless, 30-day links; add a key for permanent). Published on ClawHub for OpenClaw |
| `/pixelvault-setup` | Install CLI and configure API key |
| `/pixelvault-list` | List recently uploaded images, search them by AI description or tag, and describe an already-uploaded image (alt text, OCR, tags; needs `pixelvault-cli` 0.13.0+ for tags/describe) |
| `/pixelvault-optimize` | Compress and resize local images (no account needed): `pixelvault optimize` |
| `/pixelvault-transform` | Resize, convert, remove backgrounds, add effects or a watermark — via URL params |
| `/pixelvault-ecommerce-catalog <folder or URLs>` | Turn product photos into catalog variants (marketplace white squares, listing/PDP sizes, thumbnails, cut-outs, watermarks) and verify each one before reporting URLs |
| `/pixelvault-generate <prompt>` | Generate an image from a prompt and host it. Built-in by default (`pixelvault generate`, only your PixelVault key, monthly credits); OpenRouter for other models |
| `/pixelvault-email-images` | Host the local images in an email template (Resend, react-email) and rewrite them to permanent CDN URLs |
| `/pixelvault-desktop` | Get a local screenshot/clipboard image into a remote/cloud agent as a URL (via the PixelVault desktop app) |
| `/pixelvault-account` | Check quota/usage before bulk uploads; list, create, rotate, and revoke API keys; start a plan upgrade or open the billing portal (needs `pixelvault-cli` 0.6.0+, 0.7.0+ for upgrade/billing) |

## Quick Start

```bash
npx skills add pixelvault-dev/skill
```

Then, inside your agent:

1. Run `/pixelvault-setup` to configure your API key
2. Upload media with `/pixelvault-upload screenshot.png` or `/pixelvault-upload demo.mp4`

## Use it from any agent

- **skills.sh (any agent)** — `npx skills add pixelvault-dev/skill` installs the skill folders for Claude Code, Codex, Cursor, and more.
- **Claude Code plugin** — install from the marketplace, or copy the folders in [`skills/`](./skills) into `.claude/skills/` (project) or `~/.claude/skills/` (user).
- **OpenClaw** — `openclaw skills install pixelvault-host-image` (from [ClawHub](https://clawhub.ai)). Needs only `curl`. Guide: https://pixelvault.dev/integrations/openclaw
- **Remote MCP server** — point any MCP-capable agent at `https://mcp.pixelvault.dev/mcp` (bearer auth with your API key). No local install required.
- **Any other agent** — paste a `SKILL.md` from [`skills/`](./skills) into your custom instructions, or hand the agent the OpenAPI spec below.

## How It Works

The skill wraps the [PixelVault CLI](https://www.npmjs.com/package/pixelvault-cli).
When you or an agent uploads media:

1. The file is streamed to PixelVault's edge infrastructure
2. It is stored on Cloudflare R2
3. It is served globally through Cloudflare CDN
4. You get a permanent URL such as `https://img.pixelvault.dev/proj_abc/img_xyz.png`
   or `https://img.pixelvault.dev/proj_abc/vid_xyz.mp4`

## Requirements

- Node.js 20+
- A PixelVault account. Free tier: **200 MB storage, 500 uploads/month, 1 GB bandwidth, 5 MB per image, and 25 MB per MP4 video**. No credit card or trial expiry.

## Examples

```
> /pixelvault-upload screenshot.png
Uploaded: https://img.pixelvault.dev/proj_abc/img_xyz.png

> /pixelvault-upload *.png --folder diagrams
Uploaded: https://img.pixelvault.dev/proj_abc/img_001.png
Uploaded: https://img.pixelvault.dev/proj_abc/img_002.png

> /pixelvault-upload demo.mp4
Uploaded: https://img.pixelvault.dev/proj_abc/vid_xyz.mp4

> /pixelvault-list --json
[{ "id": "img_xyz", "url": "https://...", "size": 45200, ... }]
```

## Agent discovery

PixelVault is designed to be found and used by agents without a human in the loop:

- **MCP server** — `https://mcp.pixelvault.dev/mcp`
- **llms.txt** — [pixelvault.dev/llms.txt](https://pixelvault.dev/llms.txt)
- **OpenAPI 3.1 spec** — [api.pixelvault.dev/openapi.json](https://api.pixelvault.dev/openapi.json)
- **API catalog (RFC 9727)** — [pixelvault.dev/.well-known/api-catalog](https://pixelvault.dev/.well-known/api-catalog)

## Links

- [PixelVault](https://pixelvault.dev) — Landing page
- [API Docs](https://pixelvault.dev/docs/) — Full API reference
- [CLI](https://www.npmjs.com/package/pixelvault-cli) — CLI on npm
- [JavaScript/TypeScript SDK](https://www.npmjs.com/package/@pixelvault-dev/sdk) and [Python SDK](https://pypi.org/project/pixelvault/) — for code that calls the API directly
- [OpenAPI Spec](https://api.pixelvault.dev/openapi.json) — Machine-readable API spec

## License

MIT
