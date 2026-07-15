---
name: pixelvault-email-images
description: Host the local images referenced by an email template on PixelVault and rewrite them to permanent CDN URLs. Use when the user is building or sending an email (Resend, react-email, Nodemailer, Postmark, SendGrid) that references local image files, or says "my email images are broken", "images don't show in Gmail/Outlook", "host the images in this email", "prepare this email template for sending". Gmail doesn't render base64 `data:` images (and many Outlook clients don't either), so every image needs a public URL — this skill finds the local refs, uploads them, and swaps in the hosted URLs.
args:
  - name: template
    description: Path or glob to the email template file(s) to process (e.g. emails/receipt.tsx, ./emails/*.html)
    required: true
---

# PixelVault Email Images

Take an email template that references local images and make it send-ready:
upload each local image to PixelVault and rewrite the reference to the permanent
CDN URL it returns.

## Why this is needed

An email can only reliably render an image that lives at a public URL:

- **Base64 `data:` images don't render** in Gmail, and inconsistently in
  Outlook — recipients see a broken box.
- **CID attachments** bloat every send and hurt deliverability.
- **A hosted `https://` URL** renders everywhere. That's the target.

Resend (and every other email API) sends the message but doesn't host images, so
the images have to live somewhere public first. PixelVault URLs are permanent and
immutable on zero-egress storage — they render for the life of the inbox and cost
the same whether the email is opened once or a million times.

## Prerequisites

The `pixelvault` CLI must be installed and configured. If the command is not
found, install it:

```bash
npm install -g pixelvault-cli
```

If no API key is configured, tell the user to run `/pixelvault-setup` first or set
`PIXELVAULT_API_KEY` in their environment.

## Steps

1. **Check CLI + auth** — Run `which pixelvault`, then `pixelvault whoami --json`.
   If either fails, stop and tell the user to run `/pixelvault-setup` or set
   `PIXELVAULT_API_KEY`.

2. **Find image references** in the template(s) at `$ARGUMENTS`. Look for:
   - HTML: `<img src="...">`
   - react-email / JSX: `<Img src="..." />` — only when `src` is a **string
     literal** (`src="./logo.png"` or `src={"./logo.png"}`)
   - CSS/inline: `url(...)` in `style` / `background`
   - Inline base64 `data:image/...` URIs

   Classify each reference and handle only the ones you can safely resolve:
   - **Local file path** (relative or absolute path that exists on disk) → host
     it. Resolve relative paths against the template file's directory.
   - **Inline base64 `data:` URI** → host it via the decode step below.
   - **Non-literal `src={expr}`** — a variable, prop, or expression such as
     `src={logo}` or `src={props.image}` → **do NOT rewrite it**. There's no
     literal path to resolve, and editing it would corrupt the template. Report
     it instead so the user can host that asset and pass the URL in themselves.
   - **Already `http(s)://`, a `cid:` reference, or a 1×1 tracking pixel** → skip.

3. **Deduplicate** — the same asset (a logo) is often referenced many times.
   Upload each unique file once and reuse the URL.

4. **Upload each unique image**:

   ```bash
   # Local file on disk
   pixelvault upload <resolved-path>
   # → https://img.pixelvault.dev/proj_abc/img_xyz.png
   ```

   For an **inline base64 `data:` URI**, decode the payload to a temp file first,
   then upload that file (pick the extension from the URI's mime type):

   ```bash
   # data:image/png;base64,AAAA...  →  temp file  →  upload
   printf '%s' "<base64-payload>" | base64 -d > /tmp/pv-email-1.png
   pixelvault upload /tmp/pv-email-1.png
   ```

   The CLI prints one CDN URL per line to stdout.

5. **Rewrite the references you hosted**, replacing each local path or base64
   `data:` URI with its hosted URL. Leave the non-literal `src={expr}` references
   untouched — you only reported those. When the image has a known display
   `width`, append `?w=<2×width>&fmt=auto` for retina and keep the `width`
   attribute — e.g. `width="120"` becomes
   `src="https://img.pixelvault.dev/…?w=240&fmt=auto"`.

6. **Report** the mapping (source → hosted URL), which files you edited, and any
   non-literal `src={expr}` references you left for the user to handle.

## Output Contract

- `pixelvault upload` prints **only URLs** to stdout (one per line); human messages
  go to stderr. Use `--json` if you need the id/size/filename.
- Edit the template files in place and summarize every change — never rewrite a
  reference silently.

## Error Handling

| Error | Action |
|-------|--------|
| `command not found: pixelvault` | Tell user to `npm install -g pixelvault-cli` |
| `No API key configured` | Tell user to run `/pixelvault-setup` or set `PIXELVAULT_API_KEY` |
| `401 Unauthorized` | API key invalid — tell user to run `pixelvault login` |
| `413 Payload Too Large` | Image exceeds plan limit (5 MB free, 50 MB paid) |
| `415 Unsupported Media Type` | File is not a supported image format |
| Local path not found | Report which reference couldn't be resolved; don't rewrite it |

## Example

Given `emails/receipt.tsx`:

```tsx
import { Img } from "@react-email/components";

export function Receipt() {
  return <Img src="./assets/logo.png" width="120" alt="Acme" />;
}
```

After running the skill:

```tsx
import { Img } from "@react-email/components";

export function Receipt() {
  return <Img src="https://img.pixelvault.dev/proj_abc/img_xyz.png?w=240&fmt=auto" width="120" alt="Acme" />;
}
```

Now the template is safe to render and hand to Resend:

```ts
import { Resend } from "resend";
import { render } from "@react-email/render";
import { Receipt } from "./emails/receipt";

const resend = new Resend(process.env.RESEND_API_KEY);
await resend.emails.send({
  from: "receipts@yourdomain.com",
  to: "customer@example.com",
  subject: "Your receipt",
  html: await render(<Receipt />),
});
```

## Note on deliverability

Hosting images well makes them **render** reliably and keeps egress flat — it does
**not** affect inbox placement. Deliverability is a function of the sending domain
and SPF/DKIM/DMARC — the email provider gives you the authentication tooling, but
the DNS records and list hygiene are yours. Don't promise better open rates;
promise images that don't break.

## Related

- `/pixelvault-upload` — host individual images without the template rewrite.
- `/pixelvault-transform` — the full transform vocabulary for the hosted URLs
  (`?w=`, `?fmt=`, `?segment=`, watermarks).
