# Recipe — Verify points (buy / consume / refund / expire)

**API family:** Points: consume / refund / expire (buying points = a normal Create Invoice).

## Steps
1. **Buying** points creates a normal invoice (point priced as a service, e.g. 2.50 SAR + VAT) →
   verify via `create-invoice.md`.
2. **Consumption / refund / expiry** are separate messages — verify the received records.

## SQL side-effect probe
```sql
SELECT * FROM {{TBL_POINTS_RECIEVED_HEADERS}} WITH(NOLOCK) WHERE MSG_FROM = @ProductCode ORDER BY CREATED_ON DESC;
SELECT * FROM {{TBL_POINTS_RECIEVED_CONSUMPTION}} WITH(NOLOCK) WHERE MOINumber = @CustomerNin;
SELECT * FROM {{TBL_POINTS_RECIEVED_REFUND}} WITH(NOLOCK) WHERE MOINumber = @CustomerNin;
SELECT * FROM {{TBL_POINTS_RECIEVED_EXPIERED}} WITH(NOLOCK) WHERE MOINumber = @CustomerNin;  -- sic: EXPIERED
```
Pass = the consumption/refund/expiry record matches what the product sent.

## Edge cases to emit (critical)
- **No automatic balance check (BR-26):** the system records what the product sends **even if it
  exceeds purchased points**. This is a documented behaviour, **not a bug to invent** — call it out.
- Billing's responsibility is **proof of purchase**, not tracking later usage; reconciliation is manual.
- The future **Wallet** (central verifiable balance) is **not testable yet** → "not covered".

**Cite:** BR-26; `pricing-models.md` §3.4; `data-model.md` §5.6.
