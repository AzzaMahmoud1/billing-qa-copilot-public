# Worked Example — Dynamic invoice + real-time Sadad push (+ ZATCA)

> A full story->test-map, produced by running `method/story-analysis-method.md`.
> All identifiers here are illustrative (from the HO worked example); replace with your own.

**Story:**
> *As a product, I create a **dynamic-priced** invoice for a registered service and push it to
> **Sadad in real time**, so the customer can pay immediately.*

## Test map (lead with this — CLAUDE.md §4)

| # | Capability | API endpoint family | Table(s) `{{TBL_*}}` | Screen | SQL side-effect probe | Edge cases | Cite |
|---|---|---|---|---|---|---|---|
| TU-01 | Invoice create (Dynamic) | Create Invoice `[API SLOT]` | `{{TBL_INVOICE}}`, `{{TBL_INVOICE_DETAILS}}` | Invoices Management | header base/VAT(15%)/total consistent; line matches serviceId/Volume/UnitPrice | inconsistent Amount rejected (BR-05); UnitPrice to non-dynamic service (BR-06); VAT incl/excl (BR-10) | BR-05/06/10; `data-model.md` §5.1–5.2 |
| TU-04 | Sadad real-time upload | Push to Sadad `[API SLOT]` | `{{TBL_INVOICE_SADAD}}`, `{{TBL_SADAD_TRACK_TRANSACTION}}(_DETAILS)`, `{{TBL_INVOICE}}` | Invoices Management (upload status) | `SadadUploadStatus = Uploaded` + tracked txn + confirmation | batch path (≤24h); `NotUploaded` stuck; `DoNotUpload` (negative); retried | `glossary.md`; `data-model.md` §5.3 |
| TU-05 | ZATCA reporting (if configured) | Send to ZATCA (system-driven) | `{{TBL_INVOICE_ZATCA}}` | Invoice PDF (QR) | status in `{Reported, Cleared}` with stored response | no seq no → not sent; config off → not sent; `Error` → follow-up; tax invoice can't be cancelled (BR-13) | BR-11/12/13/14; `data-model.md` §5.4 |

> Prose and the filled template blocks below support the table; the table is the deliverable.

## Detail (filled template blocks)

**Step 0–2:** Actor = product. Outcome = invoice created + Sadad number returned + uploaded to
Sadad (`SadadUploadStatus = Uploaded`). Money/tax impact = **yes** (blocking-grade). Pricing model
= **Dynamic** (product sends Volume + UnitPrice + Amount; system validates consistency — see
`pricing-models.md` §3.1, BR-05/06). Integrations crossed = **Unified** (service must be registered),
**Sadad** (real-time push), **ZATCA** (if product configured to send — BR-11/12).

**Step 3–4 → units:**

```
### TU-01: Dynamic invoice created with consistent Volume×UnitPrice=Amount

**Test Objective:** A create-invoice request with consistent Volume, UnitPrice, Amount for a
  dynamic-configured service produces an invoice with correct base + 15% VAT + total.
**Capability / Pricing model / Integration:** Invoice create / Dynamic / Unified-registered service
**Traces to:** BR-05, BR-10

**Preconditions:**
- Product + service registered & active via Unified ({{TBL_SERVICES}}.SERVICE_ACTIVITY=1,
  service configured dynamic); customer resolvable by CustomerNin.

**Steps:**
1. Send create-invoice for the service with Volume, UnitPrice, Amount where Amount = Volume×UnitPrice.
2. Read the returned invoice number / Sadad number / amounts.

**API call(s):**
- [API SLOT — pending collection] Create Invoice: endpoint + method TBD.
  Known request fields: customerNin, serviceId, Volume, UnitPrice, Amount.
- Expected response: invoice number + Sadad number + base/VAT/total.

**SQL verification:**
- `{{TBL_INVOICE}}` — row for INVOICE_NUMBER; base amount, VAT (15%), total consistent
  (e.g. base 1000 → VAT 150 → total 1150).
- `{{TBL_INVOICE_DETAILS}}` — line matches serviceId/Volume/UnitPrice.

**UI layer:** Invoices Management — invoice is searchable by Sadad number / creation date.

**Expected result:** Invoice persisted with Amount accepted and VAT = 15% of base.

**Edge cases:** TU-02 (inconsistent Amount rejected, BR-05); TU-03 (UnitPrice on a non-dynamic
  service rejected, BR-06); VAT-inclusive vs exclusive (BR-10).

**Verdict readiness:** PASS-able via DB once create endpoint is in the collection; otherwise
  NOT-EVIDENCED.
```

```
### TU-04: Real-time push to Sadad sets SadadUploadStatus = Uploaded

**Test Objective:** Manually pushing the invoice to Sadad via API records an upload transaction
  and flips SadadUploadStatus to Uploaded.
**Capability / Integration:** Sadad real-time upload
**Traces to:** §1 Real-Time vs Batch; SadadUploadStatus

**Preconditions:** Invoice from TU-01 exists with a Sadad number; SadadUploadStatus = NotUploaded.

**Steps:**
1. Invoke the real-time Sadad push for the invoice.
2. Re-read invoice + Sadad tracking.

**API call(s):**
- [API SLOT — pending collection] Push-to-Sadad (real-time): endpoint + method TBD.
- Expected response: upload confirmation / transaction id.

**SQL verification:**
- `{{TBL_INVOICE_SADAD}}` — Sadad number present; `{{TBL_INVOICE}}.SadadUploadStatus = Uploaded`.
- `{{TBL_SADAD_TRACK_TRANSACTION_DETAILS}}` (EFFECTED_NUMBER = INVOICE_ID) + `..._TRANSACTION`
  — request/response + success message present (biller no. 085 in source examples).

**UI layer:** Invoices Management — upload status reflects Uploaded.

**Expected result:** SadadUploadStatus = Uploaded with a tracked transaction + confirmation.

**Edge cases:** batch path (auto, ≤24h); NotUploaded stuck; DoNotUpload set (negative — should not
  be pushed); push retried.

**Verdict readiness:** PASS-able via DB after push endpoint is in the collection.
```

```
### TU-05: If product configured for ZATCA, tax invoice reaches ZATCA and gets a response

**Test Objective:** When the product is configured to send to ZATCA, the invoice is sent and a
  response (Reported/Cleared) is recorded before it is treated as a tax invoice.
**Capability / Integration:** ZATCA
**Traces to:** BR-11, BR-12, BR-14

**Preconditions:** Invoice has a Sequence Number (BR-14); product config = send-to-ZATCA (cache
  refreshed after any config change, BR-12).

**Steps:**
1. Trigger/await the ZATCA send cycle (Invoice Generator / ZatcaIntegration services).
2. Read ZATCA record.

**API call(s):** [API SLOT — pending collection] (send is system-driven; may be a re-run trigger).

**SQL verification:**
- `{{TBL_INVOICE_ZATCA}}` (order by SENDING_DATE) — status in {Reported, Cleared}; Error → fail.

**UI layer:** PDF shows QR + seller/buyer + tax number (15 digits, 3…3) once Cleared.

**Expected result:** ZATCA status Cleared/Reported with a stored response.

**Edge cases:** no sequence number → not sent (waits); config off → not sent; Error → follow-up;
  tax invoice cannot be cancelled afterward (BR-13 — spin a negative TU).

**Verdict readiness:** NOT-EVIDENCED unless the evidence actually exercises a ZATCA response —
  never infer Cleared from a happy-path screen.
```

---
