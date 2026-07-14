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
3. **`jq`** and **`base64`** — standard tools used to extract and decode the image.

These are **authoring-time** credentials — they belong in your shell/`.env`, never
in a deployed site or client bundle. What ships is only the resulting CDN URL.

## Steps

1. **Pick a model.** Only image-*output* models work (OpenRouter routes image
   generation through the chat API's `modalities`). The default,
   `google/gemini-3.1-flash-image`, is fast and cheap; `google/gemini-3-pro-image`
   ("Nano Banana Pro") is higher fidelity. Slugs change — list current ones with:

   ```bash
   curl -s https://openrouter.ai/api/v1/models \
     | jq -r '.data[] | select(.architecture.output_modalities // [] | index("image")) | .id'
   ```

2. **Generate** the image and decode it to a file:

   ```bash
   curl -s https://openrouter.ai/api/v1/chat/completions \
     -H "Authorization: Bearer $OPENROUTER_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "model": "google/gemini-3.1-flash-image",
       "modalities": ["image", "text"],
       "messages": [{ "role": "user", "content": "<PROMPT>" }]
     }' \
     | jq -r '.choices[0].message.images[0].image_url.url' \
     | sed 's/^data:image\/[^;]*;base64,//' \
     | base64 -d > gen.png
   ```

   If `gen.png` is empty, the model returned no image — check the raw response for
   an `error`, and confirm the slug still emits images (step 1).

3. **Host** it on PixelVault and get the permanent URL (reuses `/pixelvault-upload`):

   ```bash
   pixelvault upload gen.png
   # → https://img.pixelvault.dev/proj_abc/img_xyz.png
   ```

4. **Report** the URL. It's permanent and globally cached. Need an OG card or a
   thumbnail? Add transform params to the same URL — no re-upload:
   `…img_xyz.png?w=1200&h=630&fmt=auto` (social card),
   `…img_xyz.png?w=700&fmt=auto&q=auto` (inline). See `/pixelvault-transform`.

## Error Handling

| Error | Action |
|-------|--------|
| `OPENROUTER_API_KEY` unset | Tell the user to get a key at openrouter.ai/keys and export it |
| Empty `gen.png` / no `images` in response | Model returned no image — verify the slug is an image-output model (step 1); inspect the response `error` |
| `404 No endpoints found for <model>` | The slug is stale or not image-capable — pick one from the step-1 list |
| `command not found: pixelvault` | `npm install -g pixelvault-cli` |
| `No API key configured` / `401` | Run `/pixelvault-setup` or set `PIXELVAULT_API_KEY` |
| `413 Payload Too Large` | Generated image exceeds the plan limit (5 MB free) — downscale, or use a paid plan |

## Notes

- **Prompts are non-deterministic.** The same prompt yields a different image each
  run. Keep the prompt in version control (frontmatter, a manifest, a comment) so
  the art is reproducible.
- **Attribute correctly.** Record which model produced an image if it matters —
  `gemini-3.1-flash-image` and `gemini-3-pro-image` are different looks.
- For OpenAI's `gpt-image-1` specifically, call OpenAI's images API directly and
  host the result with `pixelvault upload` the same way.

## Example

```bash
export OPENROUTER_API_KEY=sk-or-...
PROMPT="Flat vector blog hero, a friendly robot writing a document at a laptop, \
image flowing into a labeled vault, soft indigo gradient, no text, 16:9"

curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" -H "Content-Type: application/json" \
  -d "$(jq -n --arg m google/gemini-3.1-flash-image --arg p "$PROMPT" \
        '{model:$m, modalities:["image","text"], messages:[{role:"user",content:$p}]}')" \
  | jq -r '.choices[0].message.images[0].image_url.url' \
  | sed 's/^data:image\/[^;]*;base64,//' | base64 -d > hero.png

pixelvault upload hero.png
# → https://img.pixelvault.dev/proj_abc/img_xyz.png
```
