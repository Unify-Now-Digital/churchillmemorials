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

## Update 2 (26 Jul 2026) — opportunity de-dupe

Added a lookup so a repeat RAQ from the same contact **does not create a second open
opportunity** (the true-duplicate + design-comparison cases from the audit).

- **New module 6 — GHL API call** (`highlevel:universal`, conn `2100612` = GHL Churchill
  Memorials 2 / location OAuth): `GET /opportunities/search?location_id=JyUmi1MPcqhO5tMqwJMk
  &contact_id={{5.id}}&pipeline_id=xOo8LQfxd7iZUzkuLswI&status=open` (header `Version: 2021-07-28`).
- **Opportunity create (module 2) filter:** runs only when
  `length(6.body.opportunities) = 0` — i.e. no existing *open* opportunity for that contact
  in the 2024 pipeline.
- **Fail-open:** module 6 has a `builtin:Resume` on error, so if the lookup ever fails, the
  opportunity is still created — a genuine new lead is never silently dropped.

Behaviour:
- Blocks a 2nd open opportunity for the same contact (Bull #9903/#9904; comparison pairs).
- A returning customer whose previous opportunity was already **won/lost** is NOT blocked
  (only *open* opps count), so real new jobs still create.
- Two RAQs in the **same 15-min batch** may both slip through (GHL indexing lag) — rare.

## Update 3 (26 Jul 2026) — fix write connection (contacts were not landing)

Root cause of "leads don't appear in GHL": the two **write** modules were pointed at the
wrong GHL door.

- **Create Contact (module 5)** and **Create Opportunity (module 2)** were using connection
  `686351` — *"GHL Churchill Memorials"*, a **Company-level (Deprecated)** connection with no
  location scope. Writes through it were being rejected/misfiled — and both modules had a
  `builtin:Ignore` on error, so every run still reported **success** and the failures were
  invisible.
- The de-dupe search (module 6) was already correctly using `2100612` — the **Location OAuth**
  connection for the *Churchill Memorials* location (`JyUmi1MPcqhO5tMqwJMk`) — so contacts and
  the de-dupe lookup were operating in different places.

**Changes applied (scenario re-validated `isinvalid: false`, active, 900s schedule preserved):**
- Module 5 `__IMTCONN__`: `686351` → **`2100612`** (Location OAuth, Churchill Memorials).
- Module 2 `__IMTCONN__`: `686351` → **`2100612`**.
- Removed the `builtin:Ignore` onerror handlers from modules 5 and 2 so genuine GHL rejections
  now surface as failed runs instead of hiding behind green.
- Module 6 keeps its `builtin:Resume` (de-dupe must still fail-open).
- Confirmed compatible: `highlevel:universal` (module 6) already binds `2100612`, so the
  `highlevel` create modules accept it too.

Rollback: revert modules 5 & 2 `__IMTCONN__` to `686351` and re-add the two `builtin:Ignore`
handlers.

**Still outstanding:** the ~38 genuine July RAQs (see `backrun-recent-raqs.md`) were a dry run
and were never pushed — they need a one-off backfill into the Churchill Memorials location.

## Notes / possible follow-ups
- Opportunity value is the **website estimate and excludes the cemetery permit fee** — the
  team confirms the real quote before invoicing.
- To land RAQs at a "New Enquiry" stage instead of "Quote Sent", swap the `stageId` once
  the correct stage ID is confirmed from the 2024 pipeline.
- Data-quality watch items from the format sweep (48 quotes): 3 phone numbers are
  non-UK/short (#9978 & #9979 = same US number, #9930 = 8 digits); several first names
  carry a trailing space; #9965 last name stored as `O\'Shea`. Emails are 100% valid.
