# Coverage gaps — what only the API collection / PO can fill

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> Fidelity labels: **FACT** (stated in source) / **INFERRED** / **GAP** (needs the API collection or PO).
> Never assert a result the evidence does not show; never invent an endpoint or field.

## 7. Coverage gaps & what only the API collection / PO can fill


- **Endpoint paths, HTTP methods, request/response field names** for every operation (create invoice, push-to-Sadad, create ZATCA/Company profile, refund, credit memo, points consumption/refund/expiry, DPM subscription, operations invoice, transaction edit/exclude). The source names a few fields (`customerNin`, `contractId`, `contractItemId`, `serviceId`, `BankTransfer`, `zatcaProfileID`, `Company Profile ID`) but **no full schemas** — GAP until the Postman collection lands.
- **`{{TBL_INVOICE_QC}}` semantics** — name only; meaning INFERRED.
- **`IS_QUANTITATIVE = 2` (Accumulative)** calculation rule — not explained.
- **DPM's distinct tables/settings** — stated to differ but not enumerated.
- **Exact Client Key validation**, settings catalogue beyond `MaxExtendPeriod` / `EnableGenerateSequenceNumberForPostPaid` / `SET_REFERENCE`.
- **Financial-claim (Etimad) status values** and transition triggers.
- **Wallet** — future design, not testable yet.
- The **payment-header SQL join bug** (§5.5) should be confirmed and corrected before reuse.

## Added from QA dogfood (cycle 5, 2026-09-10)

- **Points → specific-invoice linkage is not evidenced.** The HO covers points consumption/refund/expiry
  records (`{{TBL_POINTS_RECIEVED_}}*`), but **not** a mechanism that redeems points to reduce a
  *specific invoice's* balance — reconciliation is described as manual and the central **Wallet** is a
  future design (GAP). For any "pay/reduce an invoice with points" story, the agent must answer
  **"not covered"** for the linkage and not invent it. Resolve when the API collection / PO clarifies.
