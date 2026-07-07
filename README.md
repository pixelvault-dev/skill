---
tags:
  - image hosting
  - image hosting for agents
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
# PixelVault — Image Hosting for AI Agents

Agent skill for image hosting: upload an image and get a permanent CDN URL back — from Claude Code, Codex, Cursor, or **any agent that reads `SKILL.md`**.

[PixelVault](https://pixelvault.dev) is agent-first image hosting for developers and AI coding agents. This skill lets an agent upload screenshots, diagrams, mockups, and generated images and get an instant, globally-cached CDN URL — no manual dashboard, no drag-and-drop, no copy-paste.

**When it triggers:** "screenshot this and host it", "upload this image and give me the URL", "attach this diagram to the PR", "host this mockup", or any task that produces an image needing a shareable link.

## Why PixelVault for agents

- **Just an API key** — no crypto wallet, no per-upload payment. A permanent free tier, not a trial.
- **Permanent, global URLs** — served from Cloudflare's edge CDN with zero egress fees.
- **Built for discovery** — ships a Claude Code skill, a remote **MCP server**, `llms.txt`, and an OpenAPI 3.1 spec so any agent can find and call it.
- **Works with any agent** — Claude Code, Codex, Cursor, Gemini CLI, or paste `SKILL.md` into custom instructions.

## Skills

| Skill | Description |
|-------|-------------|
| `/pixelvault-upload <file>` | Upload image(s) and get CDN URLs |
| `/pixelvault-setup` | Install CLI and configure API key |
| `/pixelvault-list` | List recently uploaded images |
| `/pixelvault-transform` | Resize, convert, remove backgrounds, add effects or a watermark — via URL params |

## Quick Start

```bash
npx skills add pixelvault-dev/skill
```

Then, inside your agent:

1. Run `/pixelvault-setup` to configure your API key
2. Upload images with `/pixelvault-upload screenshot.png`

## Use it from any agent

- **skills.sh (any agent)** — `npx skills add pixelvault-dev/skill` installs the skill folders for Claude Code, Codex, Cursor, and more.
- **Claude Code plugin** — install from the marketplace, or copy the folders in [`skills/`](./skills) into `.claude/skills/` (project) or `~/.claude/skills/` (user).
- **Remote MCP server** — point any MCP-capable agent at `https://mcp.pixelvault.dev/mcp` (bearer auth with your API key). No local install required.
- **Any other agent** — paste a `SKILL.md` from [`skills/`](./skills) into your custom instructions, or hand the agent the OpenAPI spec below.

## How It Works

The skill wraps the [PixelVault CLI](https://www.npmjs.com/package/pixelvault-cli). When you (or an agent) upload an image:

1. Image is uploaded to PixelVault's edge infrastructure
2. Stored on Cloudflare R2 (zero egress fees)
3. Served globally via Cloudflare CDN
4. You get a permanent URL like `https://img.pixelvault.dev/proj_abc/img_xyz.png`

## Requirements

- Node.js 20+
- A PixelVault account — free tier: **200 MB storage, 500 uploads/month, 1 GB bandwidth, 5 MB max file size**. No credit card, no trial expiry.

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

## Agent discovery

PixelVault is designed to be found and used by agents without a human in the loop:

- **MCP server** — `https://mcp.pixelvault.dev/mcp`
- **llms.txt** — [pixelvault.dev/llms.txt](https://pixelvault.dev/llms.txt)
- **OpenAPI 3.1 spec** — [api.pixelvault.dev/openapi.json](https://api.pixelvault.dev/openapi.json)
- **API catalog (RFC 9727)** — [pixelvault.dev/.well-known/api-catalog](https://pixelvault.dev/.well-known/api-catalog)

## Links

- [PixelVault](https://pixelvault.dev) — Landing page
- [API Docs](https://pixelvault.dev/docs) — Full API reference
- [CLI](https://www.npmjs.com/package/pixelvault-cli) — CLI on npm
- [OpenAPI Spec](https://api.pixelvault.dev/openapi.json) — Machine-readable API spec

## License

MIT
