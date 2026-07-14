---
name: pixelvault-generate
description: Generate an image with an AI model (via OpenRouter — e.g. Gemini image / "Nano Banana") and host it on PixelVault, getting a permanent CDN URL back. Use when you need to CREATE an image and then embed it somewhere durable — a blog hero, an OG/social card, a README diagram, a docs illustration — and a temporary model URL or a base64 blob won't do. Prompt in, permanent https://img.pixelvault.dev/... URL out.
args:
  - name: prompt
    description: What image to generate (e.g. "flat vector blog hero, a robot writing at a laptop, soft gradient, 16:9, no text")
    required: true
  - name: model
    description: Optional OpenRouter image-model slug (default google/gemini-3.1-flash-image)
    required: false
---

# PixelVault Generate

Generate an image with an AI model, then host it on PixelVault so what you embed
is a **permanent** URL — not a model link that expires in an hour, and not a
base64 blob you have to carry around.

```
OpenRouter (image model)  →  base64  →  PixelVault  →  https://img.pixelvault.dev/…
```

**When it triggers:** "make a hero image for this post and host it", "generate an
OG card", "create a diagram and give me a URL", or any task that needs a *new*
image at a durable link.

## Prerequisites

1. **`OPENROUTER_API_KEY`** — for image generation. Get one at
   <https://openrouter.ai/keys> and export it in the environment.
2. **PixelVault CLI, configured** — for hosting. If `pixelvault` isn't installed,
   run `npm install -g pixelvault-cli`; if no key is configured, run
   `/pixelvault-setup` or set `PIXELVAULT_API_KEY`.
3. **`jq`, `base64`, `curl`** — used to build the request safely and decode the image.

These are **authoring-time** credentials — they belong in your shell/`.env`, never
in a deployed site or client bundle. What ships is only the resulting CDN URL.

## Steps

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

## Error Handling

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

## Notes

- **Prompts are non-deterministic.** The same prompt yields a different image each
  run. Keep the prompt in version control (frontmatter, a manifest, a comment) so
  the art is reproducible.
- **Attribute correctly.** Record which model produced an image if it matters —
  `gemini-3.1-flash-image` and `gemini-3-pro-image` are different looks.
- **`MODEL` respects the `model` arg** — set `MODEL=<slug>` before running, or it
  falls back to the default.
- For OpenAI's `gpt-image-1` specifically, call OpenAI's images API directly and
  host the result with `pixelvault upload` the same way.
