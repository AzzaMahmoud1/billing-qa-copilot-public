# Invoice lifecycle, entities & identifiers

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> Fidelity labels: **FACT** (stated in source) / **INFERRED** / **GAP** (needs the API collection or PO).
> Never assert a result the evidence does not show; never invent an endpoint or field.

## 2. Entities & key identifiers


- **Invoice identifiers (do not conflate):**
  - Internal/serial **Invoice Number** (`INVOICE_NUMBER`) and surrogate **`INVOICE_ID`**.
  - **Sadad Number** shown on the document.
  - **Unified/establishment number**.
  - **Customer tax number** (15 digits, 3…3) — shown as info when available.
- **Customer record:** name, email, mobile, national address, **SAP customer number** (when available), `CustomerNin`. Linked to apps/products via **Customer Application** → multiple records per customer, each with its own **Account Number** (one record per product).
- **Pricing hierarchy:** Service → Price List → Price List Detail (COST, `SLICE_START_VOLUME`, `IS_QUANTITATIVE`, `SLICE_START_DATE`, activity flag). A service is **not** expected to bind to more than one price list, but a price list may hold multiple details (new price = new detail row with new start volume/date). Price changes are **soft-deleted**: old rows kept with `ACTIVITY=0`, new rows inserted — so historical transactions keep their reference.

### Invoice lifecycle (states & transitions)

> States named in the source: **New, Quote/Price-Offer, Tax Invoice, Paid, Refund, Cancelled, Expired**, plus **Financial Claim** (govt). ZATCA has its own status axis.

```
                         (product-config: send to ZATCA?)
  Quote / Price Offer ───────────────┐
   │  (can be cancelled)             │
   │  customer pays / seq-number     ▼
   └────────────► Tax Invoice ──► (ZATCA: New → Reported → Cleared | Error)
                     │  │
  New ───────────────┘  │  payment notification (Type 4) arrives
   │ (can be cancelled   ▼
   │  if not tax/ZATCA) Paid
   │
   │ expiry Job passes due-date
   ▼
  Expired            Refund (full/partial) ──► reversing invoice created
                     Cancelled (only pre-tax / pre-ZATCA)
```

| Axis | Values | Notes |
|---|---|---|
| **Invoice status** | New, Quote, Tax Invoice, Paid, Refund, Cancelled, Expired, Financial Claim | A Sequence Number means "tax invoice due", **not** "paid". Paid only after a **Payment Notification**. |
| **SadadUploadStatus** | `Uploaded`, `NotUploaded`, `DoNotUpload` | `NotUploaded` = not uploaded yet / action not executed. `DoNotUpload` = system decided **never** to push to Sadad. An invoice can be `New` with any of the three. |
| **ZATCA status** | `New`, `Reported`, `Cleared`, `Error` | `Cleared` = processed OK. `Error` = send failed, needs follow-up. Credit memo requires original in `Cleared`/`Reported`. |
| **Expiry** | Job flips overdue invoices to `Expired`; invoices not yet due must **not** be treated as expired. `MaxExtendPeriod` caps extension. |

---
