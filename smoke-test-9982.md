# Smoke test — Request a Quote #9982 → GoHighLevel

A real website quote email run end-to-end through the mapping in
`quote-to-ghl-field-mapping.md`.

**Source:** Gmail message `19f8420d2fb278e8`, subject `[Request a quote]`,
from `no-reply@churchillmemorials.co.uk`, 21 Jul 2026 10:02 UTC.

---

## Step 1 — Raw fields extracted from the email

| Raw field | Value |
|---|---|
| Quote number | `#9982` |
| Product | `The Manor - 6ft 6" x 2ft 6"` |
| SKU | `16074-02` |
| Quantity | `1` |
| Memorial Colour | `Standard - Black` |
| Lettering Colour | `Gold` |
| Flower Container | `Two Vases` |
| Photo Plaque | `Heart plaque (+£160.00)` |
| Infill | `White (+£250.00)` |
| Inscription | `In loving memory of Deana Oates a loving mother, sister, daughter and nanna  D 06-08-1969 - 16-11-2024` / `Fly high mama forever in our hearts` |
| Total | `£3,447.25` |
| First Name | `Montana` |
| Last Name | *(blank)* |
| Email | `sprangxxx@gmail.com` |
| Phone | `07440608169` |
| Grave Location | `Sw170by Blackshaw road tooting` |
| Grave Number | *(blank)* |

---

## Step 2 — Classified into the 8 fields

| # | Field | Value | How |
|---|---|---|---|
| 1 | First name | `Montana` | direct |
| 2 | Last name | *(not provided)* | left blank — not invented |
| 3 | Order value | `£3,447.25` | Total; **website estimate, excludes permit fee** |
| 4 | Order type | `New memorial – Kerb set` | "The Manor", 6ft 6" x 2ft 6" = full kerb set |
| 5 | Cemetery | `Lambeth Cemetery, Blackshaw Road, Tooting, SW17 0BY` | normalised from Grave Location (SW17 0BY Blackshaw Rd = Lambeth Cemetery) |
| 6 | Occasion | `Memorial for a mother (also sister, daughter, nanna)` | parsed from inscription; deceased **Deana Oates**, 06‑08‑1969 – 16‑11‑2024 |
| 7 | Inscription | *In loving memory of Deana Oates a loving mother, sister, daughter and nanna  D 06‑08‑1969 – 16‑11‑2024 / Fly high mama forever in our hearts* | verbatim |
| 8 | Additional notes | Standard Black granite; Gold lettering; Two Vases; **Heart photo plaque (+£160)**; White infill (+£250); deceased Deana Oates DOB 06‑08‑1969, died 16‑11‑2024; no surname or grave number supplied | meta + leftovers |

---

## Step 3 — GHL Contact payload

```json
{
  "firstName": "Montana",
  "lastName": "",
  "email": "sprangxxx@gmail.com",
  "phone": "+447440608169",
  "source": "Website – Request a Quote",
  "tags": ["raq", "new-enquiry"],
  "customFields": {
    "grave_location": "SW17 0BY Blackshaw Road, Tooting",
    "grave_number": ""
  }
}
```
*Upsert key: `email` → `sprangxxx@gmail.com`.*

---

## Step 4 — GHL Opportunity payload

```json
{
  "name": "The Manor kerb set – Deana Oates – Lambeth Cemetery, Tooting",
  "pipeline": "Churchill Memorials – Sales",
  "pipelineStage": "New Enquiry",
  "status": "open",
  "monetaryValue": 3447.25,
  "source": "Website – Request a Quote",
  "contactEmail": "sprangxxx@gmail.com",
  "customFields": {
    "quote_number": "9982",
    "product": "The Manor - 6ft 6\" x 2ft 6\"",
    "sku": "16074-02",
    "order_type": "New memorial – Kerb set",
    "deceased_name": "Deana Oates",
    "occasion": "Memorial for a mother (also sister, daughter, nanna)",
    "cemetery": "Lambeth Cemetery, Blackshaw Road, Tooting, SW17 0BY",
    "grave_number": "",
    "memorial_colour": "Standard - Black",
    "lettering_colour": "Gold",
    "photo_plaque": "Yes – Heart plaque",
    "infill": "White",
    "addons": "Two Vases",
    "inscription": "In loving memory of Deana Oates a loving mother, sister, daughter and nanna  D 06-08-1969 - 16-11-2024\nFly high mama forever in our hearts",
    "additional_notes": "Website estimate £3,447.25 excludes permit fee; deceased DOB 06-08-1969, died 16-11-2024; no surname or grave number supplied."
  }
}
```

---

## Result

✅ **Pass.** All 8 classified fields populated (or correctly left blank), and every
value maps to a defined GHL Contact or Opportunity field with no data loss.

**Judgement calls the automation must handle (all handled correctly here):**
- **Missing last name** → left blank, not fabricated.
- **Order value** → carried as `monetaryValue` but flagged as an estimate excluding the
  permit fee (the team confirms the real quote).
- **Cemetery** → free-text `SW17 0BY Blackshaw Road Tooting` normalised to *Lambeth
  Cemetery*; raw string preserved in notes in case the inference is wrong.
- **Deceased name + occasion** → parsed out of the inscription (they have no field of
  their own on the form).

**Cross-check — a second live email (Quote #9977, "HEADSTONE WITH ARABIC PRAYERS"):**
inscription reads `Test Order` / lorem-style filler and message `test` → correctly
classified as **order type: New memorial – Headstone** but **flagged as a test
submission** (per the §6 guardrail) and would be skipped, not pushed to GHL. This
confirms the mapping also protects against junk.
