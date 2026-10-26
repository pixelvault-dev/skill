---
name: pixelvault-host-image
description: Turn a local image file, or an image URL, into a public https link with one curl call. Use when you need a shareable URL for a screenshot, chart, generated image, or diagram, for example to send it in chat, embed it in a post, a PR, or an email. Works with no signup or API key (temporary link, 30 days); add PIXELVAULT_API_KEY for permanent links.
homepage: https://pixelvault.dev/integrations/openclaw
metadata: {"openclaw": {"emoji": "🖼️", "homepage": "https://pixelvault.dev/integrations/openclaw/", "requires": {"bins": ["curl"]}, "primaryEnv": "PIXELVAULT_API_KEY", "envVars": [{"name": "PIXELVAULT_API_KEY", "required": false, "description": "Optional PixelVault secret key (pv_live_...). Without it uploads are keyless and expire after 30 days."}]}}
---

# Host an image, get a public URL

Use this when an image only exists on this machine (a screenshot, a rendered
chart, a generated picture) and something else needs a link to it: a chat
message, a social post, a GitHub comment, an email, a web page.

Needs only `curl`. No install, no account.

## Upload a file

```bash
curl -sS -F "file=@/path/to/image.png" \
  ${PIXELVAULT_API_KEY:+-H "Authorization: Bearer $PIXELVAULT_API_KEY"} \
  https://api.pixelvault.dev/v1/images
```

The response is JSON. The link is `data.url`:

```json
{"data":{"id":"anon_10jtiv12ah9s","url":"https://img.pixelvault.dev/anonymous/anon_10jtiv12ah9s.png","mime_type":"image/png","size":70,"expires_in":2592000,"temporary":true}}
```

To print only the link: pipe the command through `jq -r .data.url`.

Reply with that URL. Open it once (`curl -sI <url>`) if the next step depends on it being reachable.

## Keyless vs keyed

| | No key | With `PIXELVAULT_API_KEY` |
|---|---|---|
| Link lifetime | 30 days (`"temporary": true`) | Permanent |
| Import from a URL | No | Yes |
| Private signed links | No | Yes |

Keyless is the default. Do not ask the user to sign up before the first upload.
If the user wants links that never expire, tell them to create a free account at
https://pixelvault.dev/dashboard/ and put the key in `PIXELVAULT_API_KEY`
(in OpenClaw: `skills.entries["pixelvault-host-image"].apiKey`).

## Import an image that already has a URL (key required)

```bash
curl -sS -X POST https://api.pixelvault.dev/v1/images \
  -H "Authorization: Bearer $PIXELVAULT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/generated.png"}'
```

Use this for short-lived URLs (generated images, expiring CDN links) before
they disappear. https only. Without a key the API returns `missing_api_key`:
download the file first, then upload it with the first command.

## Resize or convert on the fly

Append query parameters to the returned URL. No re-upload:

- `?w=800` width, `?h=600` height
- `?fmt=webp` convert (`fmt=auto` negotiates the best format)
- `?w=1200&h=630` for a social card size

## Limits and errors

- Formats: JPEG, PNG, GIF, WebP, AVIF, SVG. Max 5 MB on the free plan.
- `413` file too large, `415` not a supported image, `429` slow down and retry later.
- `401` the key is wrong. Unset it to fall back to a keyless upload.

## Rules

- Hosted images are public to anyone with the link. Do not upload screenshots
  containing secrets, tokens, or personal data. Check before uploading.
- Never print or log `PIXELVAULT_API_KEY`.
- Do not upload adult content; it is prohibited.
