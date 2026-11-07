---
name: pixelvault-ecommerce-catalog
description: Turn product photos into catalog-ready image variants on PixelVault — marketplace white-background squares, listing and product-page sizes, thumbnails, social cards, transparent cut-outs and watermarked copies — and verify each variant before reporting URLs. Use when the user says "prepare product images for Amazon/Google Shopping/our store", "make white-background versions of these products", "resize my catalog images", "re-host the supplier images", "generate thumbnails for every product", or has a folder or list of product photos that need several sizes or a clean background. Builds on /pixelvault-upload and /pixelvault-transform.
args:
  - name: source
    description: A folder or glob of product photos (e.g. ./products/*.jpg), or a list of direct https image URLs from a supplier feed
    required: true
  - name: variants
    description: Which variants to produce, e.g. "marketplace", "listing+pdp", "all". Defaults to marketplace + listing + pdp + thumb.
    required: false
---

# PixelVault E-commerce Catalog

Upload each product photo once, then derive every variant the catalog needs from its
CDN URL with query params. Nothing is re-uploaded and no image pipeline is needed. The
steps below include a **verification pass**, because PixelVault silently ignores
parameters it doesn't support and serves the original instead of an error.

## Prerequisites

- The `pixelvault` CLI installed and configured (`/pixelvault-setup`) with an API key.
  Cut-outs, effects and watermarks work only on images in a **keyed project**, not on
  anonymous (keyless) uploads.
- Check room first with `/pixelvault-account`: free is 200 MB, Starter 5 GB, Pro 25 GB.
  Estimate `products x images x average size` before a bulk run.

## Steps

1. **Ingest each photo once.**

   Local files (CLI 0.9.0+; `--dedupe` reuses an identical file instead of storing a copy):

   ```bash
   pixelvault upload ./products/1234-front.jpg --folder sku-1234 --dedupe --json
   ```

   Supplier URLs (a **direct** `https` image URL; see the notes below):

   ```bash
   curl -sS https://api.pixelvault.dev/v1/images \
     -H "Authorization: Bearer $PIXELVAULT_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"url":"https://supplier.example.com/img/1234.jpg","folder":"sku-1234","dedupe":true}'
   ```

   Both return the image `id` and `url`
   (`https://img.pixelvault.dev/proj_x/sku-1234/img_y.jpg`). **Write every SKU -> url pair
   to a mapping file as you go** (`catalog-images.json`). There is no lookup by SKU in
   the API, so this file is the only index you will have.

2. **Build the variant URLs** by appending params to the image URL:

   | Variant | Params | Result |
   |---|---|---|
   | Marketplace main image | `?segment=foreground&background=white&w=2000&h=2000&fit=pad` | AI cut-out centered on pure white, 2000x2000. Add `&fmt=webp` for a much smaller file. |
   | Listing / grid | `?w=640&h=640&fit=cover&fmt=auto` | Square crop; AVIF/WebP to browsers that support them, JPEG otherwise. |
   | Product page | `?w=1200&fmt=auto` | Larger version at its natural aspect ratio. |
   | Zoom | `?w=2000&fmt=auto` | Never exceeds the source size. |
   | Thumbnail | `?w=200&h=200&fit=cover&gravity=auto` | Cart, search and email thumbnails. |
   | Social / OG card | `?size=social` | 1200x630. |
   | Transparent cut-out | `?segment=foreground` | Transparent PNG, for compositing. |
   | Watermarked copy | `?w=1200&tile=<watermark_image_id>.png` | See the watermark note below. |

3. **Verify every variant before reporting it.** For each URL:

   ```bash
   curl -sS -o /tmp/variant -w '%{http_code} %{content_type} %{size_download}\n' "$URL"
   file /tmp/variant   # the actual format
   ```

   Check that the status is `200`, the content type is what you asked for, and, when you
   asked for a size, that the dimensions match the table (for example `sips -g pixelWidth
   -g pixelHeight /tmp/variant` on macOS, or `identify` if ImageMagick is installed).
   **A variant that is byte-for-byte the original means a parameter was ignored.** Fix the
   URL; do not report it.

   The first request for a new cut-out takes a couple of seconds, then the result is cached
   at the edge for a year. This verification request is also the cache warm-up.

4. **Spot-check cut-outs by eye.** Open one or two of the white-background results. AI
   cut-outs can misjudge glass, jewelry, fine mesh or products photographed on a similar
   color, so tell the user which products deserve a manual look.

5. **Report** a table of SKU, variant and URL, plus anything that failed verification. Do
   not claim marketplace compliance: Amazon, Google Shopping and others have their own
   current image rules. Say what was produced and let the user confirm.

## Rules that will trip you up

- **Sizes snap up to fixed steps:** 16, 32, 48, 64, 96, 128, 200, 256, 320, 400, 512, 640,
  800, 1024, 1200, 1600, 2000, 2400, 3000, 4000. Asking for 1500 returns 1600 and 1000
  returns 1024. Variants never exceed the source, so a 1200 px master cannot produce 2000.
- **A cut-out is never JPEG.** It is PNG, WebP or AVIF; asking for `fmt=jpg` returns a PNG.
- **Invalid or unsupported params are ignored.** You get the original image back, never an
  error. This is why step 3 exists.
- **`tile` takes the watermark's image id and extension** (for example `img_wm123.png`,
  prefixed with its folder if it has one), not the original upload filename. Upload the
  watermark once with the opacity baked into a PNG. It tiles edge to edge; there is no
  corner-logo placement.
- **Folders** may contain only letters, numbers, hyphens and underscores (max 64 characters),
  so use `sku-1234`, not `skus/1234`. The folder becomes part of the image URL.
- **URL import needs a direct URL.** Redirects are refused (`ssrf_blocked`: "provide a direct
  download URL"), and some hosts block server-side fetchers with a `403`. For those, download
  the file and upload it with the CLI instead.
- **Per-file size limit:** 5 MB per image on every plan. Downscale larger masters first.

## Error handling

| Error | Action |
|-------|--------|
| `ssrf_blocked` / "redirected" | The URL redirects. Resolve it to the final direct URL, or download and upload the file. |
| `fetch_failed` (status 403/404) | The host blocks fetchers or the link is dead. Download it yourself and upload the file. |
| `invalid_folder` | Remove `/` and other symbols from the folder name. |
| `413 Payload Too Large` | File is over the 5 MB cap. Downscale it. |
| `402` / quota messages | Plan limit reached. Run `/pixelvault-account` and tell the user. |
| Variant equals the original | A param was ignored. Recheck names and values against `/pixelvault-transform`. |

## Related

- `/pixelvault-upload`: upload options, private links, retries
- `/pixelvault-transform`: the full parameter reference
- `/pixelvault-account`: quota and plan before a bulk run
- Reference: <https://pixelvault.dev/docs/#transforms> and <https://pixelvault.dev/for-ecommerce/>
