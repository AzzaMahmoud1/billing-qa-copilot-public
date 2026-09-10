# Glossary — Billing & Payment domain

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> Fidelity labels: **FACT** (stated in source) / **INFERRED** / **GAP** (needs the API collection or PO).
> Never assert a result the evidence does not show; never invent an endpoint or field.

## 1. Domain glossary (EN)


| Term | Meaning in this system |
|---|---|
| **Unified / Unified Portal** | The onboarding channel where a product registers its product/service data; replaces the old manual/Jira entry. Carries `Product Code` + `Client Key`. Data flows onboarding → review/approval → QC → billing-ready. |
| **Product Code** | Identifier of the product/application. Formed from a 3-digit base + an incremental serial, e.g. base `117` → `117000001`. Must be identical across product, payment system, and billing; a code present in one system but not another **blocks** the billing cycle. Appears in SQL as `APPLICATION_ID` (examples seen: `080`, `017`, biller `085`). |
| **Client Key** | Paired with Product Code as the product↔billing link credential. (No schema detail in source — treat field-level checks as GAP.) |
| **Service ID** | Identifier of a billable service under a product; must be pre-registered before an invoice request can reference it. `{{TBL_SERVICES}}.SERVICE_ID`; external code = `SERVICE_EXTERNAL_ID`. |
| **Sadad / Sadad Number** | National bill-payment rail. After invoice creation the system may return a **Sadad Number**; the product uses it to query amount/details and pay. "Sadad Reference" = the Sadad number tied to the invoice, **not** a separate payment id. Old manual portal upload is retired; integration is direct. |
| **SadadUploadStatus** | Per-invoice status of whether the invoice was pushed to Sadad: `Uploaded`, `NotUploaded`, `DoNotUpload` (see §3). |
| **Real-Time vs Batch upload** | Real-Time = product pushes the invoice to Sadad **manually via API**. Batch = system pushes **automatically**, can take **up to 24h**. |
| **ZATCA** | Zakat, Tax and Customs Authority. Every **tax** invoice must be sent to ZATCA and its **actual response awaited** before it is registered as a tax invoice. |
| **Tax Invoice** | A confirmed financial claim on the customer; has a **Sequence Number** in Billing System and is (if configured) sent to ZATCA. Tax number pattern: **15 digits, starts and ends with `3`**. |
| **Quote / Price Offer (عرض سعر)** | A pre-payment document showing service + price + Sadad reference. Converts to a **Tax Invoice** after payment. A not-yet-sent quote **can** be cancelled; a tax invoice cannot (use credit note / refund). |
| **Financial Claim (مطالبة مالية)** | Govt-sector document raised for approval on the **Etimad (اعتماد)** platform; not a paid invoice and not a quote. Becomes a full tax invoice after approval + payment. |
| **UPSERT** | Customer data handling: if customer exists → update; else create (before invoicing). Match key = **`CustomerNin`**. |
| **CustomerNin** | The unified customer match key. Company: number typically starts **`700`**, carries VAT/CR number. Individual: national id / iqama id. Missing or mis-formatted `CustomerNin` is a known request-failure cause. |
| **Credit Memo / Credit Note (إشعار دائن)** | Reverse document linked to the original via `related invoice` / `RELEATED_INVOICE_ID`. Treated as a new invoice but references the original. Rules in §7. |
| **Refund (استرداد)** | Full or partial return of money. Amount is **entered pre-tax**; the system adds 15% VAT internally (100 → 115). Original invoice is recorded as `Refund` and a reversing invoice is created. |
| **OPM** | Operations/Product manager role that manually marks an invoice **Paid** for out-of-system payments after Finance confirms receipt. |
| **On-behalf billing (الفوترة بالإنابة)** | Company issues invoices + talks to ZATCA **on behalf of** a counterpart that cannot invoice itself (e.g. fuel stations, Hajj/Umrah hotels). Invoice shows the counterpart's tax/brand identity. |
| **Third-party / multi-party billing** | Distinct from on-behalf: a service provider party + a paying party, with the company as technical intermediary. |
| **Company Profile** | Branding/legal profile (AR/EN name, CR, address, city, postal code, logo). `Company Profile ID` is passed into product/service/invoice requests to drive the identity shown on the PDF. |
| **ZATCA Profile** | Created via "Create ZATCA Profile" using tax number + credentials; returns `zatcaProfileID` (demo value **254**) used to bind later operations to the correct tax file. |
| **DPM — Dynamic Pricing Module** | A **separate** billing path where the product sends Service ID + Volume **+ Amount**; the system does **not** recompute from the price list — it takes the amount, adds VAT, issues a subscription invoice directly. Uses **different settings/tables** from normal billing. (Do not confuse with "dynamic pricing" consistency-check in §4.) |
| **Slice / PerUnit / Accumulative** | Price-list detail calculation modes — `IS_QUANTITATIVE` = `0`=Slice, `1`=CalculatePerUnit, `2`=Accumulative (see §4). |
| **Service types** | `SERVICE_TYPE`: `1`=Subscription, `2`=Transaction, `3`=Points, `4`=SetupFees, `5`=OnePayOff (One-Off), `6`=ContractRelease. **All are sent in the `Subscriptions` section of the request except `Transaction`**, which has its own section. |
| **Non-Invoiceable service** | A service flagged non-invoiceable is treated at 0% tax and **not** sent to ZATCA as a taxable invoice. |
| **MaxExtendPeriod** | Setting capping how long an invoice's validity can be extended; may differ per product vs system level. |
| **EnableGenerateSequenceNumberForPostPaid** | Per-product setting; when on, Postpaid invoices for that product get a Sequence Number used downstream. |
| **SET_REFERENCE** | Settings are stored at Billing-System (global) level or keyed to a product/Application ID (product-specific override). `SET_REFERENCE` is used to query a specific product's settings. A **cache refresh** may be needed after changing a setting. |

---
