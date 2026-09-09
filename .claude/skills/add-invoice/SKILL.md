---
name: add-invoice
description: Add or update a wedding vendor invoice in the Wedding Budget tracker — extracts details from an invoice, updates data.json, pushes to GitHub. Use for adding an invoice, vendor, quote, or deposit.
---

# Add invoice to the Wedding Budget tracker

This skill updates `data.json` in the wedding-budget repo from an invoice, then
publishes the change so the live PWA picks it up.

## Before you start

Confirm you are in the wedding-budget repo (it should contain `index.html`,
`data.json`, `manifest.json`). If not, ask the user for the path and `cd` there.

Run `git pull` first — the user may have edited amounts from another machine.

## Step 1 — Read the invoice

The user will give you a path to a PDF or image. Extract:

- **Vendor / business name** (the trading name, not the legal entity if they differ)
- **Total amount including GST**
- **Instalment structure** — deposit vs balance, and the percentage or amount of each
- **Due dates** for each instalment
- **Invoice number / reference**
- **Anything already marked paid** on the invoice

If the invoice is ambiguous about instalments (e.g. it states a total but the
payment terms are in a separate T&Cs document), ask the user rather than guessing.
Never invent a due date — if one isn't stated, ask.

## Step 2 — Update data.json

Read the current `data.json` and match the structure exactly. Key rules:

- **`id`** must be unique, lowercase, hyphenated: `florist-deposit`, `florist-balance`
- **`payee`** must match a string in the `suppliers` array. If this is a new
  vendor, add the name to `suppliers` too.
- **`amount`** is the amount of *that instalment*, not the vendor total
- **`due`** is `YYYY-MM-DD`
- **`defaultPaid`** — only set this if the invoice shows the instalment as already
  paid. Set it to the full instalment amount. Omit the field entirely otherwise.
- **`fromLoan: true`** — only if the user says this was paid from the $25k loan
- **`ref`** — invoice number plus any terms worth remembering later

Also update:

- **`categories`** — add or adjust an entry with the vendor's *total* incl. GST.
  This drives the Spend by Category chart and is separate from the instalments.
- **`pending`** — if this vendor was previously an unquoted placeholder (e.g.
  `flowers`, `videographer`), **remove it from `pending`** now that it's a real
  quote. Leaving it in double-counts it visually.

### Example instalment entry

```json
{
  "id": "florist-deposit",
  "label": "Florist deposit (30%)",
  "payee": "Bloom & Co",
  "desc": "Deposit to secure the date — ceremony arch, bouquets, table arrangements",
  "due": "2026-11-20",
  "amount": 840.00,
  "ref": "Invoice BC-2291. Balance due 30 days prior to wedding.",
  "defaultPaid": 840.00
}
```

## Step 3 — Validate before committing

Always run this before pushing. A malformed `data.json` gives the user a blank
screen on their phone with no obvious cause:

```bash
python3 -c "import json; d=json.load(open('data.json')); \
print('OK —', len(d['instalments']), 'instalments,', len(d['suppliers']), 'suppliers')"
```

Then check every `payee` resolves to a known supplier:

```bash
python3 -c "
import json; d=json.load(open('data.json'))
bad=[i['id'] for i in d['instalments'] if i['payee'] not in d['suppliers']]
ids=[i['id'] for i in d['instalments']]
dupes=[x for x in set(ids) if ids.count(x)>1]
print('Unknown payee:', bad or 'none')
print('Duplicate ids:', dupes or 'none')
"
```

Fix anything flagged before continuing.

## Step 4 — Commit and push

```bash
git add data.json
git commit -m "Add <vendor> invoice: <short summary>"
git push
```

Use a specific commit message — `"Add Bloom & Co florist invoice: $2,800 total,
30% deposit paid"` beats `"update data"`. The git history becomes the audit trail
for what changed and when.

## Step 5 — Report back

Tell the user:

- What you added (vendor, total, instalment split, due dates)
- Anything you had to assume or couldn't determine from the invoice
- That GitHub Pages takes ~30-60 seconds to redeploy, then a refresh on their
  phone will show it

## Important caveat to pass on

The app stores the user's tick-box progress in the browser's `localStorage`.
If they have **already manually overridden** an amount or paid-figure for an
instalment in the app, their local value wins and this `data.json` change won't
appear for that specific field. New instalments and untouched fields show up fine.

If a change genuinely isn't appearing after a refresh, that's the likely cause —
the fix is for them to correct it in the app directly, not to re-edit `data.json`.
