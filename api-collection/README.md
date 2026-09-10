# API collection slot (drop-in)

The agent treats a **Postman collection as an input**, not a format it designs. Until you drop one here,
endpoints stay as **families** (`knowledge-base/endpoints.md`) and concrete calls are marked
`[API SLOT — pending collection]`. Supplying a collection **upgrades** the API column to a named,
runnable request — it never changes the DB/screen routing.

## How to add your collection

1. Export from Postman: **Collection v2.1** → save as `api-collection/collection.json`.
2. Export the matching **environment** → save as `api-collection/environment.json`
   (**gitignored** — it holds real `{{baseUrl}}`/`{{token}}` values and must never be committed).
3. Name collection **folders after KB capability ids** so the agent can join a routed story to the
   right request:

   | Folder name | Maps to recipe |
   |---|---|
   | `create-invoice` | `how-to-verify/create-invoice.md` |
   | `zatca-reporting` | `how-to-verify/zatca-reporting.md` |
   | `sadad-upload` | `how-to-verify/sadad-upload.md` |
   | `refund-and-credit-memo` | `how-to-verify/refund-and-credit-memo.md` |
   | `points-redemption` | `how-to-verify/points-redemption.md` |
   | `proxy-3rd-party-billing` | `how-to-verify/proxy-3rd-party-billing.md` |

   If folder names diverge, the agent falls back to **endpoint-family** matching — no error.

## The ingest contract (what the agent reads)

- `folder` → **capability** (keys `how-to-verify/_index.md`)
- `request` → a concrete **endpoint** (fills the API-endpoint column: name, method, path)
- saved `example` → a **positive/negative case** the agent can cite
- `{{var}}` → **environment** config (from `environment.json`)

## Files here

- `collection.placeholder.json` — an empty-but-valid v2.1 skeleton showing the expected shape. Replace
  it with your real export (or keep it alongside as a template).
- `environment.example.json` — a template with placeholder variables only. Copy to `environment.json`
  and fill real values locally (that copy is gitignored).

> **Secrets rule:** real tokens/URLs live only in the gitignored `environment.json`. Nothing secret
> enters either the private or the public repo.
