---
name: pixelvault-account
description: Check PixelVault quota/usage, manage API keys, and start an upgrade or open the billing portal. Use before a bulk upload ("do I have room for 200 images?", "how much storage is left?", "what plan am I on?"), when the user asks to "list my API keys", "rotate my PixelVault key", "revoke a key", "create a new key for CI", "upgrade my PixelVault plan", "change or cancel my subscription", or after a quota/limit error on upload.
args:
  - name: action
    description: What to do - usage, keys list, keys create, keys revoke <id>, rotate, upgrade <starter|pro>, or billing
    required: false
---

# PixelVault Account

Check quota and usage, list, create, rotate, or revoke API keys, and start a plan upgrade or open the billing portal.

## Prerequisites

Requires `pixelvault-cli` **0.6.0 or newer** for `usage` and `keys`, and **0.7.0 or newer** for `upgrade` and `billing` (`pixelvault --help` should list them). Update with `npm install -g pixelvault-cli@latest`. If the CLI is missing or unconfigured, tell the user to run `/pixelvault-setup`.

These endpoints need a **secret** key (`pv_live_...`). Publishable keys get a 403. `upgrade` and `billing` need a **live** key: `pv_test_...` keys are refused too.

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

## Upgrade the plan

```bash
pixelvault upgrade starter     # or: pro
```

Prints a Stripe Checkout URL on stdout (`--json` prints `{ "url": ... }`) and a hint on stderr. **Nothing is charged by the command: a person must open the URL and pay.** Do not try to complete payment yourself.

- Offer an upgrade only when the user asks, or when `usage` / a quota error shows they are out of room and they want to continue. Name the plan and what it unlocks, and let the user choose.
- Give the URL **only to the user** (the account owner). Do not paste it into issues, PRs, commit messages, logs, or shared channels.
- The plan changes only after Stripe confirms payment. After the user says they paid, run `pixelvault usage` and check `plan`; do not assume success.
- The account needs a verified email, and checkout is refused if a subscription already exists (use `billing` instead).

## Manage billing (change plan, payment details, cancel)

```bash
pixelvault billing
```

Prints a Stripe billing-portal URL (`--json` for `{ "url": ... }`). The user opens it to change plan, update payment details, or cancel. **That link can cancel the subscription**, so treat it like a credential: show it only to the user, never store or share it. Needs an existing subscription (`no_subscription` otherwise: use `upgrade` first).

## Limits and errors

| Situation | Meaning |
|-----------|---------|
| Max 25 keys per project | `key_limit_reached` (409): revoke an unused key first |
| 20 key changes per hour per project | `rate_limit_exceeded` (429): wait for `Retry-After` seconds |
| Revoking the key in use | `cannot_revoke_current_key` (409): switch to another key first |
| Revoking a publishable key | `not_found` (404): only secret keys are revocable |
| Publishable key used | 403: use a secret key |
| `upgrade` / `billing` with a test key | `test_key_forbidden` (403): use a live `pv_live_...` key |
| `upgrade` before verifying email | `email_not_verified` (403): verify the email first |
| `upgrade` with an active subscription | `already_subscribed` (409): use `pixelvault billing` |
| `billing` with no billing history | `no_subscription` (404): use `pixelvault upgrade` first |
| More than 10 billing requests per hour | `rate_limit_exceeded` (429): wait for `Retry-After` seconds |
| Paid plans switched off | `billing_unavailable` (503): try again later |
| Bad label/env | `validation_error` (400) |

## Raw HTTP equivalents

All at `https://api.pixelvault.dev` with `Authorization: Bearer $PIXELVAULT_API_KEY`:

```bash
curl -s -H "Authorization: Bearer $PIXELVAULT_API_KEY" https://api.pixelvault.dev/v1/usage
curl -s -H "Authorization: Bearer $PIXELVAULT_API_KEY" https://api.pixelvault.dev/v1/keys
curl -s -X POST -H "Authorization: Bearer $PIXELVAULT_API_KEY" -H "Content-Type: application/json" \
  -d '{"label":"ci","environment":"live"}' https://api.pixelvault.dev/v1/keys
curl -s -X DELETE -H "Authorization: Bearer $PIXELVAULT_API_KEY" https://api.pixelvault.dev/v1/keys/<id>
curl -s -X POST -H "Authorization: Bearer $PIXELVAULT_API_KEY" -H "Content-Type: application/json" \
  -d '{"plan":"starter"}' https://api.pixelvault.dev/v1/billing/checkout   # -> { "url": ... }
curl -s -X POST -H "Authorization: Bearer $PIXELVAULT_API_KEY" https://api.pixelvault.dev/v1/billing/portal   # -> { "url": ... }
```

Responses are wrapped in `{"data": ...}`. When piping a created key, extract `data.api_key` straight into a variable or secret store; do not print the response. Docs: <https://pixelvault.dev/docs/#usage>, <https://pixelvault.dev/docs/#keys> and <https://pixelvault.dev/docs/#billing-api>.

## Report

Summarize usage in plain terms (e.g. "12 of 500 uploads used, 189 MB storage left, resets 2026-11-01"). For keys, show ids, labels, and last-used times, never raw key values. For upgrade/billing, hand the URL to the user and say what happens next (they pay on Stripe, then `pixelvault usage` shows the new plan); never repeat the URL elsewhere.
