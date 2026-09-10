# Recipe — Verify on-behalf / third-party billing

**API family:** Create Invoice with a Company Profile. **Screen:** Invoice PDF (identity shown).

## Distinguish the two models
- **On-behalf billing (الفوترة بالإنابة):** the company issues invoices + talks to ZATCA **on behalf
  of** a counterpart that cannot invoice itself (fuel stations, Hajj/Umrah hotels). The invoice shows
  the **counterpart's** tax number / brand / logo (via Company Profile), while ZATCA integration may
  use the owning company's tax number (BR-29).
- **Third-party / multi-party:** a service-provider party + a paying party, with the company as
  technical intermediary (distinct from on-behalf).

## Steps
1. Confirm which **Company Profile** (AR/EN name, CR, address, logo) is passed into the request.
2. Create the invoice `[API SLOT — pending collection]` and download the PDF.

## SQL side-effect probe
```sql
SELECT TOP 1 * FROM {{TBL_INVOICE}} WITH(NOLOCK) WHERE INVOICE_NUMBER = @Invoice_Number ORDER BY CREATEDON DESC;
-- then inspect the rendered PDF for the displayed identity
```
Pass = the PDF shows the **counterpart's** tax/brand identity (not the technical issuer's), while
ZATCA reporting uses the configured tax file.

## Edge cases to emit
- Wrong Company Profile → wrong identity on the PDF.
- ZATCA tax number vs displayed tax number mismatch (intended for on-behalf — verify it is intentional).
- **GAP:** exact request fields for Company Profile binding — `[API SLOT — pending collection]`.

**Cite:** BR-29; `glossary.md` (On-behalf, Third-party, Company Profile); `integrations.md` (ZATCA).
