# Backfill — GHL contact IDs onto ClickUp "CM: Orders" (status `invoice sent`)

Reconciliation of ClickUp orders against GoHighLevel, keyed on the customer email
address. Ran 19 Aug 2026 against the 44 `invoice sent` tasks exported from
`CM: Orders`, all of which had an empty **GHL ID** field.

---

## 1. What was matched, and on what key

- **Source of truth for the match:** `Email (Customer)` on the ClickUp task,
  compared case-insensitively against the `email` on the GHL contact.
- **Lookup:** `GET https://services.leadconnectorhq.com/contacts/`
  with `locationId=JyUmi1MPcqhO5tMqwJMk` (Churchill Memorials) and `query=<email>`,
  API version header `2021-07-28`.
- **Acceptance rule:** write the GHL ID only when the search returned
  **exactly one** contact **and** its email matched the ClickUp email exactly.
  Anything fuzzier was left alone rather than guessed at.

Names were deliberately *not* used as a matching key — several customers are in
GHL under a maiden/married name or the name of whoever made the enquiry.

## 2. Result

| Outcome | Count |
|---|---|
| Exact single-contact match, GHL ID written | 39 |
| No contact in GHL for that email | 5 |
| Ambiguous (multiple contacts, or email mismatch) | 0 |

Field written: **GHL ID** (`26c981ce-013c-4733-893e-32ef4a0f7506`, short text) on
list `901207633256`. `ID Contact` was left untouched — on historic rows it mirrors
GHL ID, but it was out of scope here.

### 2.1 No GHL contact found (5)

These five have no contact at all in the CM location under the email on the order.
They need a contact creating, or the email on the ClickUp task is wrong.

| Task | Customer | Email on order |
|---|---|---|
| `869cpkuk5` | Dave | davesmithinbox@gmail.com |
| `869cke9nu` | Gary Clifford (Sarah Newman contribution) | gazzac1968@gmail.com |
| `869c9y6b8` | Toni Leah | Tonia.leah@aol.com |
| `869bxffpu` | Ian Ansell | ijansell@aol.com |
| `869b89xdc` | Shannon O'Callaghan | shannonocallaghan_x3@hotmail.co.uk |

### 2.2 Written but worth a human glance (2)

Both are exact email matches, so they were written — but the GHL contact name does
not obviously correspond to the ClickUp customer name.

| Task | ClickUp name | GHL contact name | Email |
|---|---|---|---|
| `869btrzf0` | Rev. Joseph Harris | nakia buchanan | revjosephharrishighschool@yahoo.com |
| `869d41e89` | New Enquiry — Arin Melvin | arin arin test | arinmelvin@gmail.com |

`869d41e89` is a test record rather than a real order.

Also noted, and believed fine: `869b3dpaf` Katherine Lockton sits in GHL as
"katherine hunter", and `869cj8ntq` Lisa Walker uses the address
`leighdupree23@gmail.com`. Both matched on email exactly.

## 3. Credential note — the n8n GHL credential is pointed at the wrong location

The n8n credential **"GHL API — Churchill Memorials"** (`McUTYbnyAV1oerN1`,
HTTP Header Auth) cannot be used for this work as it stands:

- Against the v1 host `rest.gohighlevel.com` it returns `401 "Api key is invalid."`
- Against the v2 host `services.leadconnectorhq.com` it authenticates, but every
  location-scoped call returns
  `403 "The token does not have access to this location."` — including for
  `locationId=JyUmi1MPcqhO5tMqwJMk`.

So it is a valid v2 token issued for a **different** sub-account. Anything in n8n
that depends on it for Churchill Memorials contact data is broken and will fail
the same way. It needs reissuing against the CM location.

The working path is in Make: the HTTP OAuth 2.0 connection
**"GHL Private Application (Installed on Unify & CM)"** (`13423711`) reaches the CM
location correctly. Note that the older **"CM GHL Webhook"** connection
(`10475222`) also now returns `401` and looks stale.

## 4. Reusable pieces left behind

| Where | Name | Purpose |
|---|---|---|
| Make (team 51893) | `CM: GHL Find Contact by Email` (tool `9680707`) | Read-only. Takes an email, returns matching CM GHL contacts. |
| Make (team 51893) | `CM: GHL Contact Lookup by Email (Read Only)` (scenario `9680671`) | Read-only batch version — iterator over a task/email list, returns `taskId, email, ids, emails, names` per row. Left deactivated; edit the iterator array to reuse. |

The n8n probe workflow used to diagnose the credential was archived.

## 5. Re-running for other statuses

The same routine applies unchanged to `approved`, `pending`, `form sent`,
`customer completed`, `deposit paid` and `install` — every one of those rows is
also missing a GHL ID. Swap the iterator array in scenario `9680671` for the
target task/email pairs, keep the exact-email acceptance rule, and write only the
single-match rows.
