---
name: pixelvault-host-image
description: Turn a local image file, or an image URL, into a public https link with one curl call. Use when you need a shareable URL for a screenshot, chart, generated image, or diagram, for example to send it in chat, embed it in a post, a PR, or an email. Works with no signup or API key (temporary link, 30 days); add PIXELVAULT_API_KEY for permanent links.
homepage: https://pixelvault.dev/integrations/openclaw/
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
  -H "X-PixelVault-Client: pixelvault-skill/host-image" \
  ${PIXELVAULT_API_KEY:+-H "Authorization: Bearer $PIXELVAULT_API_KEY"} \
  https://api.pixelvault.dev/v1/images
```

The response is JSON. The link is `data.url`:

```json
{"data":{"id":"anon_10jtiv12ah9s","url":"https://img.pixelvault.dev/anonymous/anon_10jtiv12ah9s.png","mime_type":"image/png","size":70,"expires_in":2592000,"temporary":true}}
```

To print only the link: pipe the command through `jq -r .data.url`.

Reply with that URL. Open it once (`curl -sI <url>`) if the next step depends on it being reachable.

The `X-PixelVault-Client` header only tells PixelVault which skill made the call (a coarse usage count). Keep it on every request.

## Embedding in a GitHub PR, issue or README

Embed a public `https://img.pixelvault.dev/...` image as `[![alt](URL)](SHARE_URL)`:
the image shows inline, and clicking it opens its share page instead of the bare
file. SHARE_URL is the URL with `https://img.pixelvault.dev/` replaced by
`https://pixelvault.dev/i/`, plus `?utm_source=share&utm_medium=markdown&utm_campaign=skill`.
Anything else (private/signed links with `?token=`, custom domains, SVG, PDF) stays
`![alt](URL)`. To print the right Markdown, pipe the upload through:

```bash
jq -r '.data.url as $u | if ($u | test("^https://img\\.pixelvault\\.dev/[^?#]+\\.(png|jpe?g|gif|webp|avif)$")) then "[![screenshot](\($u))](\($u | sub("^https://img\\.pixelvault\\.dev/"; "https://pixelvault.dev/i/"))?utm_source=share&utm_medium=markdown&utm_campaign=skill)" else "![screenshot](\($u))" end'
```

## Keyless vs keyed

| | No key | With `PIXELVAULT_API_KEY` |
|---|---|---|
| Link lifetime | 30 days (`"temporary": true`) | Permanent |
| Import from a URL | No | Yes |
| Private signed links | No | Yes |
| Change visibility, password share links | No | Yes |
| Smaller WebP/AVIF link (`data.optimized_url`) | No | Yes (public raster images) |

With a key, public raster uploads (PNG, JPEG, WebP, AVIF) also return `data.optimized_url`: the same image as WebP or AVIF, usually far smaller. Prefer it when the link goes into a page, README or email, and keep `data.url` as the original. It is absent for SVG, GIF and private uploads, so fall back to `data.url`: `jq -r '.data.optimized_url // .data.url'`.

Keyless is the default. Do not ask the user to sign up before the first upload.
If the user wants links that never expire, tell them to create a free account at
https://pixelvault.dev/dashboard/ and put the key in `PIXELVAULT_API_KEY`
(in OpenClaw: `skills.entries["pixelvault-host-image"].apiKey`).

## Import an image that already has a URL (key required)

```bash
curl -sS -X POST https://api.pixelvault.dev/v1/images \
  -H "Authorization: Bearer $PIXELVAULT_API_KEY" \
  -H "X-PixelVault-Client: pixelvault-skill/host-image" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/generated.png"}'
```

Use this for short-lived URLs (generated images, expiring CDN links) before
they disappear. https only. Without a key the API returns `missing_api_key`:
download the file first, then upload it with the first command.

## Make an uploaded image private or public (key required)

An image or PDF uploaded with a key can switch visibility in either direction.
Use its `data.id`:

```bash
curl -sS -X PATCH https://api.pixelvault.dev/v1/images/img_xyz \
  -H "Authorization: Bearer $PIXELVAULT_API_KEY" \
  -H "X-PixelVault-Client: pixelvault-skill/host-image" \
  -H "Content-Type: application/json" \
  -d '{"visibility":"private"}'
```

The response is the image, with the new link in `data.url`. A private link is
signed for 1 hour (`data.signed_expires`); get a fresh one with
`POST /v1/images/<id>/sign-url` and body `{"expires_in": 3600}`. Send
`{"visibility":"public"}` to go back. The change moves the file to a new URL:

- **Ask the user before making a public image private.** The old public URL stops
  working right away (resized versions within about a minute), so every page or
  message that embeds it breaks. Copies already saved elsewhere cannot be recalled.
- Making an image public turns off its password share links; making it private
  again does not bring them back.
- A file loses its folder on a private → public round trip.
- Same visibility as now: `200`, nothing changes.

## Password-protected share links (Starter and Pro, key required)

A share link sends one **private** image or MP4 video to a person: they open
`https://pixelvault.dev/s/shr_…`, type the password, and see it. Public files,
audio, and PDFs cannot get one. Keep the password out of the command line by
reading it from an environment variable (this needs `jq`):

```bash
jq -n '{asset_id: "img_xyz", password: env.SHARE_PASSWORD, expires_in: 604800}' |
  curl -sS -X POST https://api.pixelvault.dev/v1/share-links \
    -H "Authorization: Bearer $PIXELVAULT_API_KEY" \
    -H "X-PixelVault-Client: pixelvault-skill/host-image" \
    -H "Content-Type: application/json" \
    --data-binary @-
```

The link is `data.url` (`https://pixelvault.dev/s/shr_…`). `password` is 8–128
characters, used as sent; `expires_in` is 3600 to 7776000 seconds (1 hour to 90
days), default 7 days.

```bash
# List links, newest first (status: active, expired, revoked); ?asset_id= filters
curl -sS "https://api.pixelvault.dev/v1/share-links?asset_id=img_xyz" \
  -H "Authorization: Bearer $PIXELVAULT_API_KEY" \
  -H "X-PixelVault-Client: pixelvault-skill/host-image"

# Revoke a link now (idempotent)
curl -sS -X DELETE https://api.pixelvault.dev/v1/share-links/shr_abc123def456 \
  -H "Authorization: Bearer $PIXELVAULT_API_KEY" \
  -H "X-PixelVault-Client: pixelvault-skill/host-image"
```

- **Keep the password secret.** Never put it in chat logs, commit messages, PR or
  issue comments, logs, or anywhere public, and never next to the link. Have the
  user set it in an environment variable. PixelVault stores only a hash and never
  shows it again.
- Give the user the link and tell them to send the password to the recipient
  separately, through a different channel.

## Resize on the fly

Append `?size=<preset>` to the returned URL. No re-upload. This works on keyless
links too:

- `?size=s` (256 px wide), `?size=m` (640), `?size=l` (1280)
- `?size=social` (1200x630 card), `?size=square` (1080x1080), `?size=portrait` (1080x1350)

Free-form `?w=`, `?h=`, `?fmt=` and effects only work on links uploaded **with an
API key**. On a keyless link they are ignored and the original is served.

## Limits and errors

- Formats: JPEG, PNG, GIF, WebP, AVIF, SVG. Max 5 MB per image on the free plan and for keyless uploads, 10 MB on Starter, 15 MB on Pro.
- `413` file over your plan's cap, `400` with `invalid_file` not a supported image, `429` slow down and retry later.
- `401` the key is wrong. Unset it to fall back to a keyless upload.
- Visibility changes: `400 visibility_change_unsupported` (videos and audio keep their visibility), `409 visibility_change_collection` (file is in a collection), `409 visibility_change_conflict` (file changed meanwhile; retry), `403 visibility_change_blocked` (taken down; do not retry), `402 private_limit_reached` (Free plan's 100 private files).
- Share links: `402 share_links_require_paid_plan` (Free plan), `409 share_asset_not_private` (make it private first, with the user's consent), `400 share_asset_unsupported` (audio or PDF).

## Rules

- Hosted images are public to anyone with the link. Do not upload screenshots
  containing secrets, tokens, or personal data. Check before uploading.
- Never print or log `PIXELVAULT_API_KEY`.
- Do not upload adult content; it is prohibited.
