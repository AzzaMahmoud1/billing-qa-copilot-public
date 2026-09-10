# Recipe — Verify Sadad upload

**API family:** Push to Sadad (real-time) / Batch (auto). **Screen:** Invoices Management (upload status).

## Steps
1. Confirm the invoice has a **Sadad number**.
2. Push real-time via API `[API SLOT — pending collection]`, or wait for the batch job (auto, ≤24h).
3. Re-read invoice + Sadad tracking.

## SQL side-effect probe
```sql
-- Sadad number + per-invoice upload status
SELECT TOP 1000 * FROM {{TBL_INVOICE_SADAD}} WITH(NOLOCK)
WHERE INVOICE_ID IN (SELECT INVOICE_ID FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number);
-- upload transaction detail (EFFECTED_NUMBER = INVOICE_ID) + header (request/response/message)
SELECT TOP 1000 * FROM {{TBL_SADAD_TRACK_TRANSACTION_DETAILS}} WITH(NOLOCK)
WHERE EFFECTED_NUMBER IN (SELECT CAST(INVOICE_ID AS VARCHAR(50)) FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number)
ORDER BY TRANSACTION_ID DESC;
SELECT TOP 1000 * FROM {{TBL_SADAD_TRACK_TRANSACTION}} WITH(NOLOCK)
WHERE TRANSACTION_ID IN (SELECT TRANSACTION_ID FROM {{TBL_SADAD_TRACK_TRANSACTION_DETAILS}} WITH(NOLOCK)
  WHERE EFFECTED_NUMBER IN (SELECT CAST(INVOICE_ID AS VARCHAR(50)) FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number))
ORDER BY TRANSACTION_ID DESC;
```
Pass = `{{TBL_INVOICE}}.SadadUploadStatus = Uploaded` with a tracked transaction + success message.

## Edge cases to emit
- **`NotUploaded`** stuck = not uploaded yet / action not executed → investigate.
- **`DoNotUpload`** = system decided **never** to push — a negative: it should *not* be pushed.
- Sadad number missing → product can't pay.
- Upload transaction error / response not tracked; push **retried**.

**Cite:** `glossary.md` (Real-Time vs Batch, SadadUploadStatus); `integrations.md` (Sadad);
`data-model.md` §5.3.
