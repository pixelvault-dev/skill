---
name: pixelvault-transform
description: Transform a hosted PixelVault image via URL query params — resize, crop, convert format, remove the background (AI cut-out), apply effects (blur/grayscale/rotate), or tile a watermark. Use when the user wants a thumbnail, a specific size or format, a transparent cut-out, a grayscale or blurred version, a social/OG card, or a watermarked copy of an image already on PixelVault. No re-upload needed — it's all done by adding params to the URL.
args:
  - name: image
    description: A PixelVault CDN URL or an image ID (e.g. img_abc123)
    required: false
  - name: transform
    description: What to produce, e.g. "400px webp thumbnail", "remove background", "grayscale", "watermark with logo"
    required: false
---

# PixelVault Transform

Transform any image already hosted on PixelVault by adding query params to its CDN
URL — no re-upload, no build pipeline. Every param works on any
`https://img.pixelvault.dev/...` URL.

## Two ways to do it

**1. Append params to the URL directly** (works everywhere, no CLI needed):

```
https://img.pixelvault.dev/proj_abc/img_xyz.png?w=400&fit=cover&fmt=webp
```

**2. Use the CLI** when you have an image ID and want to print or download a variant
(requires the CLI configured — see `/pixelvault-setup`):

```bash
pixelvault get img_xyz -t "w=400&fmt=webp"                 # print the transformed URL
pixelvault get img_xyz -o thumb.webp -t "w=400&fmt=webp"   # download the variant to a file
pixelvault get img_xyz -o cut.png -t "segment=foreground"  # download a transparent cut-out
```

## Parameters

| Param | Values | Effect |
|-------|--------|--------|
| `size` | `s` \| `m` \| `l` \| `social` | Preset. `s`=256w, `m`=640w, `l`=1280w, `social`=1200×630 OG card. |
| `w`, `h` | pixels | Width/height (snapped to a discrete set for cache efficiency). |
| `fit` | `scale-down` \| `contain` \| `cover` \| `crop` \| `pad` | Resize mode. `scale-down` (default) never upscales. |
| `fmt` | `webp` \| `avif` \| `jpg` \| `png` \| `auto` | Output format. `auto` negotiates WebP/AVIF. |
| `q` | `auto` \| `60` \| `75` \| `85` | Quality. |
| `segment` | `foreground` | AI background removal → transparent cut-out. |
| `background` | `white`, `#ffaa00`, `rgba(...)` | Fill behind a removed background. |
| `gravity` | `auto` \| `face` \| `left` \| `right` \| `top` \| `bottom` | Crop gravity (with `fit=cover`/`crop`). `face` is face-aware. |
| `zoom` | `0.0`–`1.0` | Face-crop tightness (with `gravity=face`). |
| `blur` | `0`–`250` | Gaussian blur. |
| `sharpen` | `0`–`10` | Sharpen. |
| `rotate` | `90` \| `180` \| `270` | Rotate. |
| `flip` | `h` \| `v` \| `hv` | Mirror. |
| `brightness`, `contrast` | multiplier (`1` = no change) | Tone. |
| `saturation` | multiplier (`0` = grayscale, `1` = no change) | Saturation. |
| `tile` | a same-project image filename (e.g. `img_logo.png`) | Watermark — tiles one of your own images over the base. |

## Common recipes

```
?w=400                         400px-wide thumbnail
?size=social                   1200×630 social / OG card
?w=800&fit=cover&fmt=webp      800px cover-cropped WebP
?segment=foreground            background removed → transparent PNG cut-out
?segment=foreground&background=white   subject on a white fill
?saturation=0                  grayscale
?w=800&blur=30&saturation=0    blurred, grayscale 800px thumbnail
?tile=img_logo.png             tiled watermark from your own image
```

## Notes

- **Invalid params are ignored** — you always get an image back, never an error.
- **`tile`/`segment`/effects apply to your project images**, not the anonymous
  playground.
- **`tile`** must reference another **raster** image (png/jpg/webp/avif) in the
  **same project** — not an external URL. Bake opacity into the source PNG.
- SVG sources are served as-is (not transformed).
- Full reference: <https://pixelvault.dev/docs#transforms>
