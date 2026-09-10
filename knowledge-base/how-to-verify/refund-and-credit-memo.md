# Recipe — Verify refund & credit memo

**API family:** Refund / Create Credit Memo. **Screen:** Invoices Management.

## Decide the path first
- Invoice **not yet a tax invoice / not sent to ZATCA** (New or Quote) → it **can be cancelled**.
- Invoice **is a tax invoice (ZATCA sent)** → it **cannot be cancelled** (BR-13) → use **credit note /
  refund**.

## Steps
1. Establish the original invoice's state + ZATCA status (credit memo needs `Cleared`/`Reported`, BR-18).
2. For a **refund**, enter the amount **pre-tax**; the system adds 15% (BR-21; 100 → 115).
3. For a **credit memo**: on a `New` invoice the memo must equal the **full** amount due (BR-16); a
   **partial** memo is allowed only **after payment** (BR-17; base 99 → 113.85).
4. Submit `[API SLOT — pending collection]`; if the product requires **Finance approval**, follow the
   emailed approval link (BR-20).

## SQL side-effect probe
```sql
-- refund rows via the RELEATED_INVOICE_ID chain (note: column is misspelled in schema)
SELECT * FROM {{TBL_INVOICE_REFUND}} WITH(NOLOCK)
WHERE INVOICE_ID IN (
  SELECT INVOICE_ID FROM {{TBL_INVOICE}} WITH(NOLOCK)
  WHERE RELEATED_INVOICE_ID IN (SELECT INVOICE_ID FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number)
) ORDER BY 1 DESC;
-- the credit memo is a NEW invoice row linked back to the original
SELECT * FROM {{TBL_INVOICE}} WITH(NOLOCK)
WHERE RELEATED_INVOICE_ID = (SELECT INVOICE_ID FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number);
```
Pass = reversing/credit row recorded with the right pre-tax + 15% amount, linked via
`RELEATED_INVOICE_ID`; original flagged `Refund`.

## Edge cases to emit
- Partial memo (50) on a `New` invoice (due 115) → **rejected** (BR-16).
- Credit memo when ZATCA status is pending/`Error` → **rejected** (BR-18).
- Auto-approve vs Finance-approval product (BR-20).
- Govt (Etimad) transactions: **no delete** — Finance workflow only (BR-32).

**Cite:** BR-13/16/17/18/19/20/21/32; `data-model.md` §5.1.
