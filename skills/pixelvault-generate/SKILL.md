---
name: pixelvault-generate
description: Generate an image from a text prompt and host it on PixelVault, getting a CDN URL back. By default it uses PixelVault's built-in generation (one command, only your PixelVault key, Cloudflare Workers AI models, monthly credits); for a specific provider model (Gemini / "Nano Banana", etc.) it falls back to OpenRouter. Use when you need to CREATE an image and then embed it somewhere durable - a blog hero, an OG/social card, a README diagram, a docs illustration - and a temporary model URL or a base64 blob won't do. Prompt in, https://img.pixelvault.dev/... URL out.
args:
  - name: prompt
    description: What image to generate (e.g. "flat vector blog hero, a robot writing at a laptop, soft gradient, no text")
    required: true
  - name: model
    description: Optional. Built-in path - a PixelVault model id (default flux-1-schnell; sdxl-lightning, sdxl-base, lucid-origin, phoenix-1.0). OpenRouter path - an OpenRouter image-model slug (default google/gemini-3.1-flash-image)
    required: false
---

# PixelVault Generate

Generate an image, then host it on PixelVault so what you embed is a **stable
CDN URL** - not a model link that expires in an hour, and not a base64 blob you
have to carry around.

**When it triggers:** "make a hero image for this post and host it", "generate an
OG card", "create a diagram and give me a URL", or any task that needs a *new*
image at a durable link.

## Pick a path

| | **Path A: built-in (default)** | **Path B: OpenRouter** |
|---|---|---|
| Needs | Your PixelVault key (secret key, verified email) | `OPENROUTER_API_KEY` + your PixelVault key |
| One command | `pixelvault generate "<prompt>"` | several steps (generate, decode, upload) |
| Models | FLUX.1 schnell (default), SDXL, Leonardo (open Cloudflare Workers AI models) | Any OpenRouter image model, e.g. Gemini / "Nano Banana" |
| Cost | Plan credits: free 10 a month, Starter 300, Pro 1,500 | Your OpenRouter spend |
| Good at | Illustrations, backgrounds, abstract art, hero images | Whatever the chosen model is best at, e.g. text in images |

**Use Path A unless** the user names a provider or model that only Path B has, or
needs something Path A is weak at (accurate text rendered inside the image, photoreal
faces), or Path A is out of credits and the user has an OpenRouter key. If you are
unsure, try Path A first: it is one command and a failed call is refunded.

## Path A: built-in generation (default)

Needs `pixelvault-cli` 0.10.0 or newer, configured with a **secret** key whose
account has a verified email.

1. **Preflight** - fail early, before spending a credit:

   ```bash
   command -v pixelvault >/dev/null || npm install -g pixelvault-cli@latest
   pixelvault generate --help >/dev/null 2>&1 \
     || { echo "pixelvault-cli is older than 0.10.0 - run: npm install -g pixelvault-cli@latest"; exit 1; }
   pixelvault whoami --json >/dev/null 2>&1 \
     || { echo "PixelVault not authenticated - run /pixelvault-setup"; exit 1; }
   pixelvault usage   # the "AI images" line shows credits left this month
   ```

2. **Generate.** stdout is only the hosted URL; the credits used and left go to
   stderr, so capture stdout:

   ```bash
   URL="$(pixelvault generate "flat vector blog hero, a robot writing at a laptop, soft gradient, no text")"
   echo "$URL"
   # -> https://img.pixelvault.dev/proj_abc/img_xyz.jpg
   # stderr: Generated with flux-1-schnell: 1 credit used, 9 left this month.
   ```

   Options: `--model <id>`, `--folder <name>`, `--optimized` (print the WebP/AVIF
   URL, usually far smaller, best for pages and emails), `--delete-after 1h`
   (auto-delete the image), `--private` (signed link, with `--expires 12h`), `--json`
   (full response).

3. **Report** the URL. Need an OG card or a thumbnail? Add transform params to the
   same URL, no re-upload: `...img_xyz.jpg?w=1200&h=630&fit=cover&fmt=auto` (social card),
   `...img_xyz.jpg?w=700&fmt=auto&q=auto` (inline). See `/pixelvault-transform`.

### Models and credits

| Model | Credits per image | Plans |
|---|---|---|
| `flux-1-schnell` (default) | 1 | All, including Free |
| `sdxl-lightning`, `sdxl-base` (beta) | 1 | Starter, Pro |
| `lucid-origin`, `phoenix-1.0` (Leonardo) | 20 | Starter, Pro |

Credits reset on the 1st of each month (UTC). A failed generation is refunded,
deleting an image does not give its credit back, and a generated image also counts
toward your normal upload and storage limits.

### Errors (Path A)

| Error | Action |
|-------|--------|
| `command not found: pixelvault` / unknown command `generate` | `npm install -g pixelvault-cli@latest` (needs 0.10.0+) |
| `generation_credits_exhausted` (402) | Out of credits until the 1st. **Do not retry.** Tell the user, offer an upgrade, or switch to Path B |
| `model_requires_paid_plan` (402) | The free plan can only use `flux-1-schnell`; drop `--model` or upgrade |
| `email_verification_required` (403) | The account must verify its email first (`/pixelvault-setup`) |
| `generation_forbidden` (403) | A publishable key was used; generation needs a **secret** key |
| `prompt_rejected` (422) | Prompts that clearly ask for adult content are refused; reword |
| `rate_limit_exceeded` (429) | 60 generations per hour per account, or the day's capacity: wait and retry later |
| `generation_failed` / `generation_unavailable` / `generation_timeout` (502/503/504) | The credit is refunded; safe to retry once |
| `401` / not authenticated | Run `/pixelvault-setup` or set `PIXELVAULT_API_KEY` |

The first result is a draft: models do not follow every instruction literally, so
re-run or reword the prompt when it matters. Keep the prompt in version control so
the art is reproducible. Each model has its own license, so check the terms before
using output commercially (Cloudflare's model catalog lists them).

## Path B: OpenRouter (other models)

Use this when you need a provider model Path A doesn't have.

```
OpenRouter (image model)  ->  base64  ->  PixelVault  ->  https://img.pixelvault.dev/...
```

### Prerequisites

1. **`OPENROUTER_API_KEY`** — for image generation. Get one at
   <https://openrouter.ai/keys> and export it in the environment.
2. **PixelVault CLI, configured** — for hosting. If `pixelvault` isn't installed,
   run `npm install -g pixelvault-cli`; if no key is configured, run
   `/pixelvault-setup` or set `PIXELVAULT_API_KEY`.
3. **`jq`, `base64`, `curl`** — used to build the request safely and decode the image.

These are **authoring-time** credentials — they belong in your shell/`.env`, never
in a deployed site or client bundle. What ships is only the resulting CDN URL.

### Steps

Run these in order. Each step fails loudly so an agent never proceeds with a
missing key or a broken file.

1. **Preflight** — verify tools + auth *before* spending a generation call:

   ```bash
   : "${OPENROUTER_API_KEY:?set OPENROUTER_API_KEY — https://openrouter.ai/keys}"
   for t in curl jq base64 pixelvault; do
     command -v "$t" >/dev/null || { echo "missing required tool: $t"; exit 1; }
   done
   pixelvault whoami --json >/dev/null 2>&1 \
     || { echo "PixelVault not authenticated — run /pixelvault-setup"; exit 1; }
   ```

2. **Pick a model.** Only image-*output* models work (OpenRouter routes image
   generation through the chat API's `modalities`). The default,
   `google/gemini-3.1-flash-image`, is fast and cheap; `google/gemini-3-pro-image`
   ("Nano Banana Pro") is higher fidelity. Slugs change — list current ones with:

   ```bash
   curl -fsS https://openrouter.ai/api/v1/models \
     | jq -r '.data[] | select(.architecture.output_modalities // [] | index("image")) | .id'
   ```

3. **Generate.** Build the request body with `jq` so the prompt is always
   JSON-safe (quotes, apostrophes, and newlines in art prompts won't break it),
   fail on HTTP errors (`-fsS`), and extract the image with `jq -er` so a missing
   image is caught instead of silently decoding to garbage:

   ```bash
   MODEL="${MODEL:-google/gemini-3.1-flash-image}"
   PROMPT="a flat vector blog hero, a robot writing at a laptop, soft gradient, 16:9, no text"

   body="$(jq -n --arg m "$MODEL" --arg p "$PROMPT" \
     '{model:$m, modalities:["image","text"], messages:[{role:"user", content:$p}]}')"

   resp="$(curl -fsS https://openrouter.ai/api/v1/chat/completions \
     -H "Authorization: Bearer $OPENROUTER_API_KEY" \
     -H "Content-Type: application/json" \
     -d "$body")" || { echo "OpenRouter request failed"; exit 1; }

   # -e makes jq exit non-zero when the path is absent/null (i.e. an error response)
   data_url="$(printf '%s' "$resp" | jq -er '.choices[0].message.images[0].image_url.url')" \
     || { printf '%s\n' "$resp" | jq -r '.error.message // "no image in response"'; exit 1; }

   # Decode to a temp file (no clobbering the CWD) and confirm it's non-empty
   img="$(mktemp -t pv-gen).png"
   printf '%s' "$data_url" | sed 's/^data:image\/[^;]*;base64,//' | base64 -d > "$img"
   test -s "$img" || { echo "decoded image is empty — aborting"; exit 1; }
   ```

   > `base64 -d` reads stdin on both GNU and macOS/BSD. If an older BSD build
   > rejects `-d`, use `-D`.

4. **Host** it on PixelVault (reuses `/pixelvault-upload`):

   ```bash
   pixelvault upload "$img"
   # → https://img.pixelvault.dev/proj_abc/img_xyz.png
   ```

5. **Report** the permanent URL. Need an OG card or a thumbnail? Add transform
   params to the same URL — no re-upload: `…img_xyz.png?w=1200&h=630&fmt=auto`
   (social card), `…img_xyz.png?w=700&fmt=auto&q=auto` (inline). See
   `/pixelvault-transform`.

### Error handling

| Error | Action |
|-------|--------|
| `OPENROUTER_API_KEY` unset | Get a key at openrouter.ai/keys and export it |
| `curl` fails / non-200 from OpenRouter | Inspect `$resp` for `.error.message`; check the key and the model slug |
| jq `-e` exits non-zero (no image in response) | Model returned no image — verify the slug is image-output (step 2); read the printed error |
| `404 No endpoints found for <model>` | Slug is stale or not image-capable — pick one from the step-2 list |
| decoded image empty (`test -s` fails) | Decode failed — on older BSD `base64`, use `-D`; re-check the response |
| `command not found: pixelvault` | `npm install -g pixelvault-cli` |
| PixelVault `401` / not authenticated | Run `/pixelvault-setup` or set `PIXELVAULT_API_KEY` |
| PixelVault `413 Payload Too Large` | Image exceeds the plan limit (5 MB free) — downscale, or use a paid plan |
| PixelVault `415 Unsupported Media Type` | Decoded file isn't a supported image — the generation step likely failed |

### Notes

- **Prompts are non-deterministic.** The same prompt yields a different image each
  run. Keep the prompt in version control (frontmatter, a manifest, a comment) so
  the art is reproducible.
- **Attribute correctly.** Record which model produced an image if it matters —
  `gemini-3.1-flash-image` and `gemini-3-pro-image` are different looks.
- **`MODEL` respects the `model` arg** — set `MODEL=<slug>` before running, or it
  falls back to the default.
- For OpenAI's `gpt-image-1` specifically, call OpenAI's images API directly and
  host the result with `pixelvault upload` the same way.
