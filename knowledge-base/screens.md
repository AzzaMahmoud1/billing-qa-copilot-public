# Screens catalogue (UI WHERE)

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> UI surfaces named in the handover. Use these for the **UI layer** of a test map — what the tester can
> *observe*. Exact field labels / layout beyond what the HO states are **GAP**.

| Screen | What it shows / does | Role-gated? | Use it to verify | Known issues |
|---|---|---|---|---|
| **Invoices Management** | Search invoices by **Sadad number / creation date / status**; view; **download**; **cancel**; **update payment status** (mark Paid). | **Yes** — view/control differ by role; admins have wider rights. | Invoice is searchable and in the expected state; upload status reflects `Uploaded`; payment status updated with reference number. | — |
| **Invoice / Quote PDF** (`DownloadInvoice`) | Renders invoice or quote to PDF. Tax invoice shows invoice no., tax number (15 digits, `3…3`), VAT, total, QR; quote shows different fields. | — | Document content (seller/buyer, tax number, VAT, QR) once ZATCA `Cleared`; AR/EN rendering. | **Redis cache bug (BR-31):** may return the stale copy — description change was a temporary workaround. Financial values are **not editable** from this screen (BR-30). |
| **Unified service-entry** (onboarding) | Product/service registration: `Product Code` + `Client Key`, AR/EN name, `SERVICE_TYPE`, VAT%, pricing model, non-invoiceable flag; review → approval → QC. | Yes | Service registered + active before it can be invoiced (BR-01). | — |
| **Credit-memo approval link** | Emailed approval link for products that require **Finance approval** of a credit memo; approver opens link → approves → state changes. | Yes (approver) | The approval workflow transitions the memo to approved and processing completes (BR-20). | — |
| **Settings** (product vs system level) | View/edit settings e.g. `MaxExtendPeriod`, `EnableGenerateSequenceNumberForPostPaid`, send-to-ZATCA; keyed by `SET_REFERENCE`. | Yes | A config change took effect — remember a **cache refresh** may be needed (BR-12). | — |

> Rule of thumb (from the method): **API proves the request was accepted, DB proves it was done right,
> UI proves it is visible to the user.** A money/tax/state story needs the DB-layer check to be
> PASS-able; the screen confirms visibility, not correctness.
