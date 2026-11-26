---
name: pixelvault-transform
description: Transform a hosted PixelVault image via URL query params, custom named presets, or HMAC-signed private transform URLs — resize, Retina DPR, auto-trim, flatten animated GIF/WebP, convert format, remove the background (AI cut-out), apply effects (blur/grayscale/rotate), add a tiled or positioned watermark, or render a text overlay. Use when the user wants a thumbnail, Retina asset, trimmed product shot, transparent cut-out, grayscale or blurred version, social/OG card, watermarked/labeled copy, reusable named preset (`?size=<name>`), strict preset-only mode, or a transformed signed URL for a private image.
args:
  - name: image
    description: A PixelVault CDN URL or an image ID (e.g. img_abc123)
    required: false
  - name: transform
    description: What to produce, e.g. "400px webp thumbnail", "2x retina avatar preset", "trim whitespace", "watermark bottom-right", "signed 800px private preview"
    required: false
---

# PixelVault Transform

Transform any image already hosted on PixelVault by adding query params to its CDN
URL — no re-upload, no build pipeline. Public images work directly via URL query
params on `https://img.pixelvault.dev/...`; private images use server-signed
transform URLs (`POST /v1/images/:id/sign-url` or `pixelvault sign-url`).

> Working with a **local file** that isn't hosted yet (compress, resize, convert before committing)? Use `/pixelvault-optimize` instead — it runs on the user's machine with no account.

## Prerequisites (for CLI commands)

- Public URL transforms need **no CLI**.
- `pixelvault get -t` works on any recent CLI.
- `pixelvault presets` (custom named presets + strict mode) and `pixelvault sign-url` (signed transforms for private images) require `pixelvault-cli` **0.29.0 or newer** (`npm install -g pixelvault-cli@latest`) and a **secret** API key.

## Ways to do it

**1. Append params to a public URL directly** (works everywhere, no CLI needed):

```
https://img.pixelvault.dev/proj_abc/img_xyz.png?w=400&fit=cover&dpr=2&fmt=auto
```

**2. Use the CLI** when you have an image ID and want to print or download a variant:

```bash
pixelvault get img_xyz -t "w=400&fmt=webp"                 # print the transformed URL
pixelvault get img_xyz -o thumb.webp -t "w=400&fmt=webp"   # download the variant to a file
pixelvault get img_xyz -o cut.png -t "segment=foreground"  # download a transparent cut-out
```

**3. Mint a signed transform URL for a private image** (CLI 0.29.0+):

```bash
pixelvault sign-url img_private --expires 1h -t "w=800&fmt=auto&watermark=img_logo.png"
pixelvault sign-url img_private --expires 30m --params '{"size":"card","dpr":2}' --json
```

> **Never append query params to an already-signed URL.** The HMAC signature covers both expiry and transform parameters; modifying the query string invalidates the link (`403`). Always mint a new signed URL via `pixelvault sign-url` or `POST /v1/images/:id/sign-url`.

## Parameters

| Param | Values | Effect |
|-------|--------|--------|
| `size` | Built-in (`s` \| `m` \| `l` \| `social` \| `square` \| `portrait`) or custom preset name | Named preset. Built-ins: `s`=256w, `m`=640w, `l`=1280w, `social`=1200×630, `square`=800×800, `portrait`=800×1000. |
| `w`, `h` | pixels (1–4000) | Width/height (snapped to a discrete set for cache efficiency). |
| `fit` | `scale-down` \| `contain` \| `cover` \| `crop` \| `pad` | Resize mode. `scale-down` (default) never upscales. |
| `fmt` | `auto` \| `webp` \| `avif` \| `jpg` \| `png` | Output format. `auto` negotiates AVIF/WebP via `Accept`. |
| `q` | `auto` \| `60` \| `75` \| `85` | Quality step. |
| `dpr` | `1`–`3` (snapped to `1`, `1.5`, `2`) | Retina device pixel ratio multiplier (with `w` or `h`). |
| `trim` | `border` (or `auto`) | Auto-trim uniform outer whitespace/borders before resizing. |
| `anim` | `false` \| `true` | `anim=false` flattens an animated GIF/WebP to its first frame. |
| `metadata` | `none` \| `copyright` \| `keep` | EXIF/C2PA retention (`none` strips metadata by default). |
| `segment` | `foreground` | AI background removal → transparent cut-out. |
| `background` | `white`, `%23ffaa00` (hex — percent-encode the `#`), `rgba(...)` | Fill behind a removed background. |
| `gravity` | `auto` \| `face` \| `left` \| `right` \| `top` \| `bottom` | Crop gravity (with `fit=cover`/`crop`). `face` is face-aware. |
| `zoom` | `0.0`–`1.0` | Face-crop tightness (with `gravity=face`). |
| `blur` | `0`–`250` | Gaussian blur. |
| `sharpen` | `0`–`10` | Sharpen. |
| `rotate` | `90` \| `180` \| `270` | Rotate. |
| `flip` | `h` \| `v` \| `hv` | Mirror. |
| `brightness`, `contrast` | multiplier (`1` = no change) | Tone. |
| `saturation` | multiplier (`0` = grayscale, `1` = no change) | Saturation. |
| `tile` | `image_id.ext` of a same-project public raster image | Tiled watermark across the entire image. |
| `watermark` | `image_id.ext` of a same-project public raster image | Single positioned watermark overlay. |
| `watermark_pos` | `bottom-right` (default) \| `bottom-left` \| `top-right` \| `top-left` \| `center` | Corner/center placement for `watermark`. |
| `watermark_opacity` | `0`–`1` | Opacity for `watermark`. |
| `watermark_width` | pixels (`1`–`2000`) | Target width for `watermark`. |
| `text` | UTF-8 string (max 120 chars; percent-encode spaces/symbols) | Text overlay badge rendered over the image. |
| `text_pos` | `bottom-right` (default) \| `bottom-left` \| `top-right` \| `top-left` \| `center` | Placement for `text`. |
| `text_color` | CSS color / hex (percent-encode `#` as `%23`) | Foreground color for `text`. |
| `text_size` | `8`–`200` (px) | Font size for `text`. |
| `text_opacity` | `0`–`1` | Opacity for `text`. |

## Common recipes

```
?w=400                                                   400px-wide thumbnail
?size=social                                             1200×630 social / OG card
?w=800&fit=cover&dpr=2&fmt=auto                          Retina 2x cover-cropped AVIF/WebP
?trim=border&w=800&fmt=auto                              Auto-trim whitespace, then resize
?anim=false&w=400&fmt=webp                               Static first-frame thumbnail of an animated GIF/WebP
?segment=foreground                                      Background removed → transparent PNG cut-out
?segment=foreground&background=white                     Subject on a white fill
?saturation=0                                            Grayscale
?tile=img_logo.png                                       Tiled watermark from your own image
?watermark=img_logo.png&watermark_pos=bottom-right&watermark_opacity=0.6&watermark_width=140
                                                         Positioned corner watermark
?text=DRAFT&text_pos=center&text_color=%23ffffff&text_size=48&text_opacity=0.7
                                                         Centered text watermark
```

## Custom named presets & strict mode (CLI 0.29.0+)

Define reusable project-scoped presets (`5` on Free, `25` on Starter, `100` on Pro)
and optionally enable `strict_transforms` (all plans) to reject arbitrary ad-hoc
transform query params on public URLs with `403`:

```bash
# Create or update a custom preset (slug: 2-40 lowercase a-z, 0-9, -, _)
pixelvault presets set avatar "w=256&h=256&fit=cover&gravity=face&dpr=2&fmt=auto"
pixelvault presets set product-card --params '{"trim":"border","w":600,"fmt":"auto"}'

# Apply it on any project image URL:
# https://img.pixelvault.dev/proj_abc/img_xyz.png?size=avatar

# List presets, plan capacity, and strict-mode state
pixelvault presets list
pixelvault presets list --json

# Enable or disable strict transform mode (blocks ad-hoc ?w=, ?blur=, etc. with 403)
pixelvault presets strict on
pixelvault presets strict off
pixelvault presets strict        # prints current state (on/off)

# Delete a custom preset
pixelvault presets delete avatar
```

- Custom presets cannot reference `size` (no recursion) and cannot shadow built-in names (`s`, `m`, `l`, `social`, `square`, `portrait`).
- With `strict_transforms` enabled, public URLs only accept declared custom presets and built-ins (`?size=<name>`), while server-signed private transform URLs still work.
- Raw HTTP equivalents (`Authorization: Bearer $PIXELVAULT_API_KEY`):
  - `GET /v1/presets`
  - `PUT /v1/presets/:name` with `{"params":{...}}`
  - `DELETE /v1/presets/:name`
  - `PATCH /v1/projects/:id` with `{"strict_transforms":true}`
  - `POST /v1/images/:id/sign-url` with `{"expires_in":3600,"transforms":{...}}`

## Notes

- **Invalid params on public URLs are ignored** — you always get an image back, never an error (unless `strict_transforms` is enabled, in which case ad-hoc params return `403`).
- **Percent-encode reserved characters** when building a URL by hand — especially `#` (`%23`), spaces (`%20`), and parens/commas in `rgba(...)`. The CLI (`pixelvault get -t`, `pixelvault sign-url -t`) handles encoding for you.
- **`tile`, `watermark`, `text`, `segment`, and effects apply to project images**, not the anonymous playground.
- **`tile` and `watermark`** must reference another **public raster** image (png/jpg/webp/avif) in the **same project** by its `image_id.ext` (prefix its folder if it has one) — never a private `cp/...` path, upload filename, or external URL.
- **A cut-out (`segment`) is PNG, WebP or AVIF — never JPEG.** `fmt=jpg` returns a PNG.
- **Private signed transforms** work on raster images only — PDFs return `document_transforms_unsupported` and SVGs return `svg_transforms_unsupported`. Responses are never placed in the shared public CDN cache.
- SVG sources on public URLs are served as-is (not transformed).
- Full reference: <https://pixelvault.dev/docs/transforms/> and <https://pixelvault.dev/docs/api/transform-presets/>
