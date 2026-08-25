# DuroMax / DuroStar Parts Catalog — Customer-Facing

A public, no-login parts reference and order-request tool for DuroMax and DuroStar generator owners. Customers browse exploded parts diagrams by model, build an order list, and submit it as a parts inquiry — no NetSuite access, no account, no pricing shown.

**This is a separate, standalone project from the internal `duromax-parts-diagrams` service-desk tool.** They share no code, no repo, and no data pipeline at runtime — this repo's `index.html` embeds its own sanitized copy of the parts data with all NetSuite fields (internal IDs, direct inventory links) stripped out entirely.

## What's in this repo

- `index.html` — the entire app: layout, styles, and logic in one file, with the parts catalog (`PARTS`) and series/model map (`SERIES`) embedded inline as JS constants. No build step, no server — this is what GitHub Pages serves directly.
- `assets/` — DuroMax logo, DuroStar logo, homepage hero banner
- `images_hotspot/<MODEL>/fig<N>.jpeg` — primary per-figure diagram images (same convention as the internal tool)
- `images/<MODEL>/<page>.jpeg` — raw full-page fallback images, used only if a figure's cropped image isn't available yet

## How the data got here

The parts data originates from the internal tool's `parts_data_current.json`. Before anything is copied into this repo, it goes through a strip step:

- **Removed entirely:** `id` (NetSuite internal ID), `u` (NetSuite item URL)
- **Kept:** model, figure number/title/page, ref #, part #, description, qty-used-in-assembly
- **Added:** `a` (1 or 0) — whether this row has a real part number and should show "Add to Cart," vs. rows with a genuine manufacturer `(Pending Part #)` gap, which show "Contact us for this part" instead

There is intentionally no live connection between the two repos. When a new model is added on the internal side, the same sanitize-and-copy step needs to be repeated manually to bring it over here — see "Keeping this in sync" below.

## Order flow: how a submission actually gets to you

1. Customer adds parts to their cart (stored in their browser's `localStorage`, so it survives a refresh but is private to their device — nothing is stored server-side).
2. They click **Request This Order**, fill in name/email/phone (optional)/note, and submit.
3. The app tries **EmailJS** first (see setup below). If EmailJS isn't configured yet, or the request fails for any reason, it automatically falls back to opening a pre-filled `mailto:` link addressed to `orderparts@duromaxpower.com` — the customer just hits send from their own email app. Either path shows the same confirmation screen with an order reference number (format `PR-YYYYMMDD-####`) for follow-up.

### Setting up EmailJS (recommended — do this to skip the mailto: fallback)

Right now `EMAILJS_CONFIG.CONFIGURED` is set to `false` near the top of the `<script>` block in `index.html`, so every order currently goes out via the `mailto:` fallback. To switch to direct email delivery:

1. Create a free account at [emailjs.com](https://www.emailjs.com/).
2. Add an **Email Service** — connect the mailbox you want orders to be sent *from* (a Gmail account, Outlook, or SMTP credentials for `orderparts@duromaxpower.com` itself if that mailbox exists). Note the **Service ID**.
3. Create an **Email Template** with a "To" address of `orderparts@duromaxpower.com` and a "Reply-To" of `{{reply_to}}`. The template can reference these variables, which the app sends with every order:
   - `{{order_ref}}`, `{{customer_name}}`, `{{customer_email}}`, `{{customer_phone}}`, `{{customer_note}}`, `{{order_summary}}`, `{{line_count}}`, `{{qty_total}}`
   Note the **Template ID**.
4. Grab your **Public Key** from Account → API Keys.
5. In `index.html`, find this block near the top of the `<script>` section and fill it in:
   ```js
   const EMAILJS_CONFIG = {
     CONFIGURED: true,                     // flip this to true
     PUBLIC_KEY: 'your_public_key_here',
     SERVICE_ID: 'your_service_id_here',
     TEMPLATE_ID: 'your_template_id_here',
   };
   ```
6. Commit and push. The public key is meant to be embedded client-side like this — EmailJS rate-limits by domain, not by keeping the key secret.

`orderparts@duromaxpower.com` is hardcoded as the recipient (`ORDER_TO_EMAIL` constant, and the template's "To" field) — it's never shown to or editable by the customer.

## Keeping this in sync with the internal tool

There's no automation for this yet — it's a manual step at the start of a session, same spirit as the internal tool's own handoff-package workflow:

1. Pull the latest `parts_data_current.json` and `series_current.json` from the internal project (or export a fresh one from the live internal app).
2. Strip `id`/`u`, add the `a` flag, as described above.
3. Splice the resulting rows into this repo's `index.html` `PARTS`/`SERIES` constants.
4. Copy any newly-added `images_hotspot/<MODEL>/` and `images/<MODEL>/` folders over.
5. Validate (Node syntax check, row/figure counts, Playwright smoke test) the same way the internal tool does, then push.

## What this deliberately does NOT do

- No pricing or live stock display — this is a reference + quote-request tool, not a storefront
- No payment processing — order requests are inquiries, confirmed manually by your team
- No customer accounts or login
- No connection, direct or indirect, to NetSuite
