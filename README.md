# pixelvault-skill

An agent skill that lets AI coding assistants upload and host images via [PixelVault](https://pixelvault.dev). Zero config, no auth required.

## Install

```bash
npx skills add pixelvault/pixelvault-skill
```

## What It Does

Gives your AI agent the ability to upload images to PixelVault's playground API and get back a CDN URL — in one command. No API key needed.

- **2 MB** max file size
- **1 hour** expiry (playground mode)
- **10 uploads/hour** rate limit
- Supports JPEG, PNG, GIF, WebP, AVIF

## Usage

After installing, just ask your agent:

> "Upload this screenshot to PixelVault"

> "Host this image and give me the URL"

> "I need a CDN link for this PNG"

The agent will use `curl` to upload the file and return the CDN URL.

## For Permanent Hosting

Playground uploads expire after 1 hour. For permanent image hosting, sign up for a free account at [pixelvault.dev](https://pixelvault.dev).

## Links

- [PixelVault](https://pixelvault.dev) — API-first image hosting for developers
- [Playground API Docs](https://docs.pixelvault.dev) — Full API reference
- [skills.sh](https://skills.sh) — Agent skills directory

## License

MIT
