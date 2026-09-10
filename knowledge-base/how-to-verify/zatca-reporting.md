# Recipe — Verify ZATCA tax reporting

**API family:** Send to ZATCA (system-driven; may expose a re-run trigger). **Screen:** Invoice PDF (QR).

## Steps
1. Confirm the invoice has a **Sequence Number** (BR-14) and the product is configured to send to ZATCA
   (BR-12; a **cache refresh** may be needed after changing the setting).
2. Trigger/await the send cycle (Invoice Generator / ZatcaIntegration services).
3. Read the ZATCA record.

## SQL side-effect probe
```sql
SELECT TOP 1000 * FROM {{TBL_INVOICE_ZATCA}} WITH(NOLOCK)
WHERE INVOICE_ID IN (SELECT INVOICE_ID FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number)
ORDER BY SENDING_DATE DESC;
```
Pass = status in `{Reported, Cleared}` with a stored response. `Error` = fail → follow up.

## Edge cases to emit
- No sequence number → **not sent** (waits) — not a pass.
- Config off → not sent; non-invoiceable service → 0% tax, not sent (BR-27).
- **Stuck waiting** for response → during testing a re-run can speed the response.
- **Never infer `Cleared` from a happy-path screen** — require the actual response row (CLAUDE.md §5, rule 3: NOT-EVIDENCED is never a silent PASS).
- A **tax invoice cannot be cancelled** after ZATCA send (BR-13) → spin a negative unit.

**Cite:** BR-11/12/13/14/27; `integrations.md` (ZATCA); `data-model.md` §5.4.
