# Recipe — Verify invoice creation

**API family:** Create Invoice (`endpoints.md`). **Screen:** Invoices Management (`screens.md`).

## Steps
1. Pin the **pricing model** (`pricing-models.md`): Dynamic sends `Volume`+`UnitPrice`+`Amount`;
   Fixed/Slice/PerUnit send `Volume` only; DPM sends `Amount` (no recompute).
2. Ensure preconditions: product + service registered & active (BR-01), customer resolvable by
   `CustomerNin` (BR-03).
3. Send create-invoice `[API SLOT — pending collection]`; capture invoice number + Sadad number +
   base/VAT/total.

## SQL side-effect probe
```sql
-- header: amount / VAT(15%) / total / status / linkage
SELECT TOP 1 * FROM {{TBL_INVOICE}} WITH(NOLOCK)
WHERE INVOICE_NUMBER = @Invoice_Number ORDER BY CREATEDON DESC;
-- line items match serviceId / Volume / price
SELECT * FROM {{TBL_INVOICE_DETAILS}} WITH(NOLOCK)
WHERE INVOICE_ID IN (SELECT INVOICE_ID FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number);
-- subscription lines (period / volume / value)
SELECT * FROM {{TBL_INVOICE_SERVICE_SUBSCRIPTION}} WITH(NOLOCK)
WHERE INVOICE_ID IN (SELECT INVOICE_ID FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number);
```
Pass = header amounts consistent with the pricing model + VAT = 15% of base (e.g. 1000 → 150 → 1150).

## Edge cases to emit
- **Dynamic:** inconsistent `Amount` must be rejected (BR-05); `UnitPrice` to a non-dynamic service is
  an error (BR-06).
- **Fixed:** subscription volume not matching the price list is an error (BR-07); amount computed
  against an inactive/old `{{TBL_PRICE_LIST_DETAILS}}` row.
- **Slice:** band boundary at `SLICE_START_VOLUME`; system must apply flat band price, **not** multiply
  (BR-08).
- **PerUnit:** wrong band at boundary; high-volume rounding drift; VAT-inclusive vs exclusive (BR-09/10).
- **Paid:** a Sequence Number ≠ Paid; invoice becomes Paid only on a Payment Notification Type 4
  (BR-15) → verify `{{TBL_PAYMENT}}` + `data-model.md` §5.5.

**Cite:** BR-01/03/05/06/07/08/09/10/15; `data-model.md` §5.1–5.2.
