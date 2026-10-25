---
name: pixelvault-account
description: Check PixelVault quota/usage and manage API keys. Use before a bulk upload ("do I have room for 200 images?", "how much storage is left?", "what plan am I on?"), when the user asks to "list my API keys", "rotate my PixelVault key", "revoke a key", or "create a new key for CI", or after a quota/limit error on upload.
args:
  - name: action
    description: What to do - usage, keys list, keys create, keys revoke <id>, or rotate
    required: false
---

# PixelVault Account

Check quota and usage, and list, create, rotate, or revoke API keys.

## Prerequisites

Requires `pixelvault-cli` **0.6.0 or newer** (`pixelvault --help` should list `usage` and `keys`). Update with `npm install -g pixelvault-cli@latest`. If the CLI is missing or unconfigured, tell the user to run `/pixelvault-setup`.

These endpoints need a **secret** key (`pv_live_...`). Publishable keys get a 403.

## Check usage (do this before bulk uploads)

```bash
pixelvault usage          # plan, period, used / limit / remaining
pixelvault usage --json   # the API `data` object
```

Reports storage and uploads against the plan limits, when the period resets, and the monthly bandwidth **limit**. Bandwidth usage is intentionally not reported. Usage is scoped to the key's project.

Before uploading N files, compare `remaining.uploads` and `remaining.storage_bytes` to what you are about to send. If there is not enough room, tell the user instead of starting a partial batch.

## List keys

```bash
pixelvault keys list
pixelvault keys list --json
```

Shows id, type (`secret` or `publishable`), environment, label, key prefix, created/last-used times, and which key is `current` (the one making the request). Raw keys and hashes are never returned.

## Create a key

```bash
KEY=$(pixelvault keys create --label ci --env live)
```

- stdout is **only the raw key**; the "shown once" warning goes to stderr.
- `--label` max 60 chars; `--env` is `live` or `test` (default live). `--json` prints the whole data object (id, api_key, type, environment, label, created_at).
- The raw key is shown **once** and cannot be retrieved later. Store it in a secret manager or env var immediately. Never echo it, log it, print it in chat, or commit it.

## Revoke a key

```bash
pixelvault keys revoke <id>
```

Only **secret** keys can be revoked (a publishable key id returns not found). The key making the request cannot revoke itself.

## Rotate a key

1. Create the replacement: `NEW=$(pixelvault keys create --label rotated)`
2. Switch to it: `export PIXELVAULT_API_KEY="$NEW"` (and update wherever the old key is stored, e.g. CI secrets).
3. Find the old key's id: `pixelvault keys list` (the old one is no longer `current`).
4. Revoke it using the new key: `pixelvault keys revoke <old-id>`.

The self-revoke guard is best-effort. If you lock yourself out, the user can recover with **Regenerate** in the dashboard.

## Limits and errors

| Situation | Meaning |
|-----------|---------|
| Max 25 keys per project | `key_limit_reached` (409): revoke an unused key first |
| 20 key changes per hour per project | `rate_limit_exceeded` (429): wait for `Retry-After` seconds |
| Revoking the key in use | `cannot_revoke_current_key` (409): switch to another key first |
| Revoking a publishable key | `not_found` (404): only secret keys are revocable |
| Publishable key used | 403: use a secret key |
| Bad label/env | `validation_error` (400) |

## Raw HTTP equivalents

All at `https://api.pixelvault.dev` with `Authorization: Bearer $PIXELVAULT_API_KEY`:

```bash
curl -s -H "Authorization: Bearer $PIXELVAULT_API_KEY" https://api.pixelvault.dev/v1/usage
curl -s -H "Authorization: Bearer $PIXELVAULT_API_KEY" https://api.pixelvault.dev/v1/keys
curl -s -X POST -H "Authorization: Bearer $PIXELVAULT_API_KEY" -H "Content-Type: application/json" \
  -d '{"label":"ci","environment":"live"}' https://api.pixelvault.dev/v1/keys
curl -s -X DELETE -H "Authorization: Bearer $PIXELVAULT_API_KEY" https://api.pixelvault.dev/v1/keys/<id>
```

Responses are wrapped in `{"data": ...}`. When piping a created key, extract `data.api_key` straight into a variable or secret store; do not print the response. Docs: <https://pixelvault.dev/docs/#usage> and <https://pixelvault.dev/docs/#keys>.

## Report

Summarize usage in plain terms (e.g. "12 of 500 uploads used, 189 MB storage left, resets 2026-11-01"). For keys, show ids, labels, and last-used times, never raw key values.
