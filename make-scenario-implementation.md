# Implementation record — RAQ → GHL enrichment (Make)

**Applied 23 Jul 2026** by editing the existing live scenario **in place** (no new
scenario, no duplicate leads).

- **Make scenario:** `CM: WooCommerce > GHL` (id `1017361`, team `51893`, folder
  *Churchill Memorials*)
- **Trigger:** WooCommerce *Watch Orders* (conn `1468569`) every 15 min — the website RAQ
  arrives as WooCommerce order metadata `_raq_request` (firstName, lastName, email, Phone,
  GraveLocation, InscriptionText, message). This is why we build on WooCommerce, not on
  parsing the Gmail HTML.
- **GHL connection:** `686351` (unchanged). Pipeline `xOo8LQfxd7iZUzkuLswI` ("2024"),
  stage `c49d33ea-8b61-4935-aa4e-967434295495` ("Quote Sent"), assignee
  `9qSERogT3YZZSLczKh3K` (Arin) — all unchanged.

## What changed

**Contact (`highlevel:createAContact`)** — three previously-empty custom fields now mapped:

| GHL custom field | Field ID | Now mapped from |
|---|---|---|
| Cemetery & Plot | `TmidNAekKnYbu0mjsylb` | `_raq_request` → GraveLocation |
| Deceased Name | `ndnImMtTwbDoi2wqo4z6` | `_raq_request` → InscriptionText |
| Your Message | `BrjnBAW8RrOHFF2sQOy8` | `_raq_request` → message |

(Already mapped, unchanged: Memorial SKU/Name `EAIfZruyI58HFNvNtkTE`, Enquiry Type
`YINidnwnhYBWTmllOQyM` = "New Memorial", tags `new memorial` + `raq`, source `website`.)

**Opportunity (`highlevel:createAnOpportunity`)**:
- `monetaryValue` → **`{{1.total}}`** (full website order total / estimate) — was line-item price.
- `title` → **`{{5.firstName}} {{5.lastName}} — {{1.lineItems[].sku}}`** — was blank billing name + SKU.

**Test/junk guard** — added to the contact-module filter, so these are skipped:
- `message` equals `test`, or
- `InscriptionText` contains `Test Order`.

## Verified
Blueprint re-fetched after save: `isinvalid: false`, scenario active, schedule preserved
(indefinitely / 900s). Changes confirmed present in modules 5 and 2.

## Notes / possible follow-ups
- Opportunity value is the **website estimate and excludes the cemetery permit fee** — the
  team confirms the real quote before invoicing.
- Same-customer design-comparison RAQs (see `backrun-recent-raqs.md`) still create one
  opportunity each; collapsing those to a single opportunity would need a GHL
  search-contact step before create — not included in this edit.
- To land RAQs at a "New Enquiry" stage instead of "Quote Sent", swap the `stageId` once
  the correct stage ID is confirmed from the 2024 pipeline.
