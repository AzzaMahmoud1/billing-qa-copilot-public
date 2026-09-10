# Business rules & error conditions (BR-01..BR-34)

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> Fidelity labels: **FACT** (stated in source) / **INFERRED** / **GAP** (needs the API collection or PO).
> Never assert a result the evidence does not show; never invent an endpoint or field.

## 6. Business rules & error conditions (evidenced)


| ID | Rule | Evidence | Test as |
|---|---|---|---|
| **BR-01** | A service referenced in an invoice request must be **pre-registered** (Service ID) and active, else the request fails. | lines 236–242, 318 | negative: unregistered/inactive Service ID → reject |
| **BR-02** | Product Code must be **consistent across product, payment, and billing**; missing in one → billing cycle blocked. | line 32 | negative: code in one system only |
| **BR-03** | Customer is **UPSERTed by `CustomerNin`**; record created before invoicing if absent. | lines 26, 459 | positive + idempotency: same Nin twice updates, not duplicates |
| **BR-04** | Missing or mis-formatted `CustomerNin` (or placeholder instead of real value) → request fails. | lines 205, 245 | negative + boundary (format) |
| **BR-05** | **Dynamic pricing:** product sends Volume+UnitPrice+Amount; system validates Amount consistency before issuing; may need Finance approval. | line 14 | negative: inconsistent amount must be rejected |
| **BR-06** | **`UnitPrice` sent to a service not configured as dynamic** is an error. | line 93 | negative |
| **BR-07** | **Subscription volume not matching the price list** is an error. | line 93 | negative |
| **BR-08** | **Slice:** apply the band's flat price, **no multiplication** by quantity. | line 18 | boundary at band edges |
| **BR-09** | **PerUnit:** amount = Volume × band per-unit price. | line 20 | positive + boundary |
| **BR-10** | VAT 15% added to the model-computed base; product price may be VAT-inclusive or exclusive and must be handled accordingly. | lines 22, 319–325 | positive: inclusive vs exclusive flag |
| **BR-11** | Every **tax** invoice must be sent to ZATCA and its **actual response awaited** before registering as tax invoice. | line 8, 104 | integration: wait-for-response |
| **BR-12** | ZATCA send is **controlled by product config**; a cache refresh may be needed after changing it. | line 102 | config scenario |
| **BR-13** | A **tax invoice cannot be cancelled directly** after ZATCA send; use credit note / refund. Only a not-yet-sent quote can be cancelled. | lines 106, 111, 176 | negative: cancel tax invoice → blocked |
| **BR-14** | Invoice needs a **Sequence Number** to be eligible for ZATCA; no seq number → waits for payment/seq. | lines 224, 509 | state check |
| **BR-15** | Sequence Number ≠ Paid. Invoice becomes **Paid only on a Payment Notification** (Type 4). | lines 471, 481 | state machine |
| **BR-16** | **Credit memo on a `New` invoice must equal the full amount due** — a partial (50 on 115) is rejected. | line 115, 180 | negative |
| **BR-17** | **Partial credit memo only allowed after payment** (base 99 → 113.85 with VAT). | line 115 | positive after payment |
| **BR-18** | Credit memo requires the original invoice's ZATCA status to be **`Cleared` or `Reported`**; pending/Error → rejected. | line 186 | negative |
| **BR-19** | Credit memo is a **new invoice linked to the original** via `related invoice` / `RELEATED_INVOICE_ID`. | lines 119, 184 | linkage check |
| **BR-20** | Some products auto-approve credit memos; others require **Finance approval** (email → approver link → state change). | lines 117, 182 | workflow |
| **BR-21** | **Refund amount is entered pre-tax; system adds 15%** (100 → 115). Original flagged `Refund`, reversing invoice created. | lines 48, 113, 178 | positive |
| **BR-22** | **Out-of-system payment:** Finance confirms receipt → OPM manually marks Paid with reference number + method; system does not auto-verify the bank. | lines 40–42 | workflow |
| **BR-23** | **Expiry Job** flips overdue invoices to `Expired`; not-yet-due invoices must not be treated as expired; `MaxExtendPeriod` caps extension. | lines 201–203 | Job + boundary (due-date) |
| **BR-24** | **Invalid future date** / **server time skew (~3h)** causes invoice-date errors. | line 205 | boundary (date) |
| **BR-25** | **Anti-re-billing:** usage transactions stamped with `invoiceId` are excluded from re-billing the same period. | line 463 | idempotency |
| **BR-26** | **Points have no automatic balance check** — system records what the product sends even if it exceeds purchased points. | line 378 | negative (documented gap, not a bug to file) |
| **BR-27** | **Non-Invoiceable service** → 0% tax, not sent to ZATCA as taxable. | line 526 | config scenario |
| **BR-28** | VAT rate comes from product/service config (0/5/15%); a service is non-taxable only if **configured** so, not by changing the request value. | lines 522, 524 | negative: request VAT override ignored |
| **BR-29** | **On-behalf billing:** invoice shows the counterpart's tax number/brand/logo (Company Profile) while ZATCA integration may use the owning company's tax number. | lines 410, 437, 451 | document-content check |
| **BR-30** | `DownloadInvoice` renders invoice/quote to PDF; **financial values are system-imposed and not editable from the download screen**; fields (e.g. service name) hideable by config. | lines 108, 128, 195 | document check + Redis-cache bug (BR-31) |
| **BR-31** | **Redis cache bug:** download may return the stale copy; description change was a temporary workaround. | line 495 | regression |
| **BR-32** | **Financial Claim (Etimad):** starts as claim (not paid, not quote) → tax invoice after approval + payment; govt transaction deletion is **forbidden** (Finance workflow only). | lines 487–493 | workflow + negative |
| **BR-33** | **DPM:** product sends amount; system adds VAT and issues subscription invoice directly **without** recomputing from the price list. | lines 399, 403 | positive + negative (VAT double-add) |
| **BR-34** | Service routing: all service types sent in `Subscriptions` **except `Transaction`** (own section). | line 221 | request-shape check |

---
