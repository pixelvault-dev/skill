---
name: pixelvault-optimize
description: Compress and resize local image files before they are committed, attached, emailed or uploaded — no account or API key needed. Use when the user says "compress this image", "make these screenshots smaller", "optimize the images in this folder", "convert these PNGs to WebP", "this image is too big", or before adding images to a repo or docs. Runs on the user's machine and never writes a file bigger than the original. For images already hosted on PixelVault, use /pixelvault-transform instead.
args:
  - name: files
    description: Image file(s) or a directory to optimize (e.g. hero.png, ./assets)
    required: true
  - name: options
    description: Optional, e.g. "webp", "max width 1600", "in place"
    required: false
---

# PixelVault Optimize

Shrink local images with `pixelvault optimize`. It runs on the user's machine: **no account, no API key, nothing is uploaded** (unless `--host` is passed). It resizes, converts and strips metadata, and it never writes a file bigger than the original.

## Prerequisites

The `pixelvault` CLI **0.14.0 or newer** must be installed:

```bash
npm install -g pixelvault-cli      # or: npx pixelvault-cli optimize ...
pixelvault --version               # must be 0.14.0+
```

`optimize` uses `sharp`, an optional dependency. If the command says it can't load `sharp`, reinstall the CLI without `--omit=optional` (`npm install -g pixelvault-cli`). It needs Node 20.9+.

## Steps

1. **Preview first** — see what would change without writing anything:

```bash
pixelvault optimize ./assets -r --dry-run
```

2. **Run it.** Pick the form that matches the request:

```bash
# Default: writes hero.optimized.png next to the original (original untouched)
pixelvault optimize hero.png

# Convert + cap the width, into a separate folder (mirrors subfolders with -r)
pixelvault optimize ./shots -r --format webp --max-width 1600 --out-dir ./optimized

# Overwrite the originals (same format only) — only when they are under version control
pixelvault optimize ./assets -r --in-place

# Machine-readable report (always a JSON array)
pixelvault optimize ./shots -r --json

# Optimize, upload, and print the CDN URL (needs an API key; see /pixelvault-setup)
pixelvault optimize hero.png --host
```

3. **Report the saving** from stderr (per file and total) and tell the user which files were written, and which were skipped and why.

## Options

| Flag | Meaning |
|------|---------|
| `--format keep\|webp\|avif\|jpg\|png` | Output format. Default `keep`. WebP is usually the smallest for web use |
| `--max-width N` / `--max-height N` | Downscale to at most this size in px. Never upscales |
| `--quality 1-100` | JPEG/WebP/AVIF quality, default 80. PNG output is lossless |
| `--out-dir DIR` | Write results into DIR |
| `--in-place` | Overwrite each original (only with `--format keep`) |
| `-r`, `--recursive` | Descend into directories |
| `--dry-run` | Report savings, write nothing |
| `--keep-metadata` | Keep EXIF/GPS (stripped by default) |
| `--host` | Upload the result and print its CDN URL |
| `--json` | JSON report |

## Behaviour to know

- **Safe by default.** Without `--in-place` the original is never touched. The command refuses to run if two inputs would write to the same output, or an output would overwrite an input.
- **Never bigger, and idempotent.** A file that doesn't get smaller is left alone ("already optimal"), and re-running over already-optimized files does nothing, so it is safe in hooks and CI.
- **Metadata is stripped** (EXIF, GPS), but orientation is applied first so photos stay the right way up. Use `--keep-metadata` only if the user needs it.
- **Supported:** JPG, PNG, WebP, AVIF. GIF, SVG, animated images and HEIC are skipped (HEIC: https://pixelvault.dev/tools/heic-to-url/).
- A corrupt or oversized (over 100 MP / 100 MB) image is reported as a failure and the command exits 1.

## Choosing settings

- **Photos:** `--format webp --quality 80`, and `--max-width 1600` (or the width they are actually displayed at).
- **Screenshots and graphics (PNG):** `--format webp` gives the biggest saving; `--format keep` keeps lossless PNG.
- **Already in a repo:** prefer `--in-place` so the diff replaces the original instead of adding a second file, but confirm the files are committed first.

## Images already hosted on PixelVault

Don't download and re-compress them. Ask for a compressed variant through the CDN instead (see `/pixelvault-transform`), for example `?fmt=webp&q=75&w=1600`, or upload with `pixelvault upload photo.png --optimized` to get the smaller WebP/AVIF URL.

## Error handling

- **"needs the optional 'sharp' package"** → reinstall the CLI without `--omit=optional`; check Node is 20.9+.
- **"would both be written to ..."** → two inputs map to one output (e.g. `a.png` and `a.jpg` with `--format webp`). Optimize them separately or use `--out-dir`.
- **"output would overwrite the original"** → `--out-dir` points at the input's own folder; use a different folder or `--in-place`.
- **`--host` needs an API key** → run `/pixelvault-setup`.
