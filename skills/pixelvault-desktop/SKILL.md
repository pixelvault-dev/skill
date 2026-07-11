---
name: pixelvault-desktop
description: Get a screenshot or clipboard image from the user's local machine into THIS agent as a URL, when the agent runs remotely (cloud, SSH, sandbox like Daytona/e2b, or any text-only input) and can't accept a pasted/attached image or reach a local file. Guides the user to the PixelVault desktop app — press ⇧⌘2 to capture (or copy any image) → a hosted URL lands on the clipboard → paste it here. Use when the user says "here's a screenshot", "let me show you my screen", "how do I share an image with you", "I can't paste an image", or a pasted image didn't come through.
---

# Get a local image into this agent (as a URL)

This agent can read an image from a **URL** even when it can't accept a pasted or
attached image, and even when the image only exists on the user's screen or
clipboard (not on a filesystem this agent can reach). The **PixelVault desktop
app** turns any local image into a hosted URL on the clipboard, ready to paste.

## When to use this vs. `/pixelvault-upload`

- **This skill** — the image is on the user's **screen or clipboard**, and this
  agent is **remote** (cloud / SSH / sandbox) so it can't see the user's local
  files. The user pastes a URL.
- **`/pixelvault-upload`** — the image is a **file this agent can access** (a
  path in the current session). Upload it directly via the CLI instead.

## Guide the user

1. **If the app is installed** — tell the user to either:
   - Press **⇧⌘2** to capture a screen region (macOS native crosshair), or
   - Copy any image to the clipboard (e.g. **⇧⌃⌘4** for a region on macOS).

   The app uploads it and shows an **"Image URL copied"** notification. That's
   the cue — the hosted URL is now on the clipboard.

2. **Then paste the URL here** (⌘V) so this agent can fetch the image.

3. **If the app is NOT installed** — point the user to it:
   - Download: <https://github.com/pixelvault-dev/desktop/releases>
   - It's a small macOS/Linux menu-bar app (free & open source). No install is
     needed on this remote side — it runs on the user's local machine and only
     ever hands over a URL.

## Notes

- **Wait for the notification.** Uploads take ~0.5–2s; if the user pastes before
  the "Image URL copied" toast, they'll paste the raw image (or nothing), not the
  URL.
- **Anonymous by default** — 5 free uploads, images expire in ~30 days. Signing
  in (in the app's Settings) gives permanent-account uploads + history.
- Once you have the URL, treat it like any image URL — fetch/analyze it as needed.
