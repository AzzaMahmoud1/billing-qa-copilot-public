# Endpoint families (API WHERE)

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> The HO names **no full endpoint paths or request/response schemas** — so every concrete path, method,
> and field below is a **GAP** marked `[API SLOT — pending collection]`. What *is* evidenced is the set
> of **operations** (endpoint families) and a few **field names** the source mentions. When a Postman
> collection is dropped into `api-collection/`, each family upgrades to a concrete, named, runnable
> request. **Never invent a path or field** — leave the slot marker.

## How to read this

- **Family** = the operation the story needs (stable, evidenced).
- **Concrete request** = `[API SLOT — pending collection]` until the collection is wired.
- **Known fields** = field names actually mentioned in the HO (quoted); everything else is a GAP.
- **Verify at** = the DB table(s) in `data-model.md` that prove the operation did the right thing.

| Endpoint family | Capability | Known request fields (from HO) | Verify side-effect at (`data-model.md`) | Notes / rules |
|---|---|---|---|---|
| **Create Invoice** | invoice-create (subscription/transaction) | `customerNin`, `serviceId`, `Volume`, `UnitPrice`, `Amount`; `Subscriptions[]` section; `Transaction` has its own section (BR-34) | `{{TBL_INVOICE}}`, `{{TBL_INVOICE_DETAILS}}`, `{{TBL_INVOICE_SERVICE_SUBSCRIPTION}}` | Returns invoice number + Sadad number + base/VAT/total + expiry. Pricing model decides inputs (see `pricing-models.md`). |
| **Create Invoice (DPM)** | dynamic-pricing-module | `serviceId`, `Volume`, `Amount` (no recompute) | `{{TBL_INVOICE}}` (+ DPM tables = GAP) | System adds VAT and issues a subscription invoice directly (BR-33). |
| **Push to Sadad (real-time)** | sadad-upload | `[API SLOT]` | `{{TBL_INVOICE_SADAD}}`, `{{TBL_INVOICE}}.SadadUploadStatus`, `{{TBL_SADAD_TRACK_TRANSACTION}}(_DETAILS)` | Manual API push; Batch path is auto (≤24h). `Uploaded`/`NotUploaded`/`DoNotUpload`. |
| **Query Sadad bill** | sadad-query | Sadad number | `{{TBL_INVOICE_SADAD}}` | Product queries amount/details before paying. |
| **Payment Notification (inbound)** | payment | operation no., bank-transfer no., bank code, channel, method, time, amount, RRN, card digits | `{{TBL_PAYMENT}}`, `{{TBL_PAYMENT_SADAD}}`, `{{TBL_PAYMENT_NOTIFICATION_QUEUE}}`, `TBL_OUTBOX_MESSAGE` | Type **4**. Flips invoice to **Paid** (BR-15). Check processed + retry. |
| **Mark Paid (out-of-system)** | payment (manual/OPM) | reference number, payment method (`BankTransfer` etc.) | `{{TBL_INVOICE_HISTORY}}`, `{{TBL_PAYMENT}}` | OPM marks Paid after Finance confirms (BR-22); no auto bank check. |
| **Send to ZATCA** | zatca-reporting | system-driven (may expose a re-run trigger) | `{{TBL_INVOICE_ZATCA}}` (order by `SENDING_DATE`) | Needs Sequence Number (BR-14); status `New→Reported→Cleared/Error` (BR-11/12). |
| **Refund (full/partial)** | refund | refund amount **pre-tax** (+15% added) | `{{TBL_INVOICE_REFUND}}`, `{{TBL_INVOICE}}` (`RELEATED_INVOICE_ID`) | BR-21; partial only after payment (BR-17). |
| **Create Credit Memo / Credit Note** | refund/credit-memo | links to original via `related invoice` / `RELEATED_INVOICE_ID` | `{{TBL_INVOICE}}` (new row + `RELEATED_INVOICE_ID`) | BR-16/18/19/20; needs ZATCA `Cleared`/`Reported`; may need Finance approval. |
| **Cancel Invoice** | cancel | `[API SLOT]` | `{{TBL_INVOICE_HISTORY}}` | Only pre-tax / pre-ZATCA (BR-13). |
| **Download Invoice (PDF)** | download | invoice/quote id | n/a (render) | Financial values not editable from screen (BR-30); Redis stale-copy bug (BR-31). |
| **Create ZATCA Profile** | onboarding/tax | tax number + credentials → returns `zatcaProfileID` (demo 254) | GAP | Binds later ops to the tax file. |
| **Register Product / Service (Unified)** | onboarding | `Product Code` + `Client Key`; service: AR/EN name, `SERVICE_TYPE`, VAT%, pricing model, non-invoiceable flag | `{{TBL_APPLICATIONS}}`, `{{TBL_SERVICES}}`, `{{TBL_APPLICATIONS_SERVICES}}`, `{{TBL_SERVICE_PRICE_LIST}}` | Onboarding → review → approval → QC (BR-01/02). |
| **Points: consume / refund / expire** | points-redemption | `ConsumedPoints`, `ConsumedAmount`, `MOINumber`, `ServiceId`, `RefundPoints`, `ExpiredPoints` | `{{TBL_POINTS_RECIEVED_}}*` (headers/consumption/refund/expired) | **No balance validation** (BR-26) — call out, never invent. |
| **Usage transaction edit/exclude** | postpaid-operations | `contractId`, `contractItemId`, `serviceId`, Volume, pre-tax amount, transaction date | `{{TBL_INVOICE_SERVICE_TRANSACTION}}` | Anti-re-billing stamp (BR-25); availability in prod = GAP. |

## When the collection arrives

For each family above, the collection's matching **folder** (named after the capability id) supplies the
concrete request; a saved **example** becomes a citable positive/negative case. Until then, the agent
outputs the **family + verify-at table** and marks the concrete call `[API SLOT — pending collection]`.
