# Story-Analysis Method

> This is the **repeatable procedure** the QA-copilot agent runs on a **new** user story about
> the B2 New Payment Portal billing/APIs. It turns a story into atomic, testable units and tells
> the tester **where** to test each one: **API layer / DB layer / UI layer**.
>
> It reads from `../knowledge-base/` (`glossary.md`, `invoice-lifecycle.md`, `pricing-models.md`,
> `integrations.md`, **`data-model.md`**, `business-rules.md`, `endpoints.md`, `screens.md`) and the
> `how-to-verify/` recipes. Cite `BR-*` / table names from there.
>
> **Hard rule (inherited from CLAUDE.md §3):** never assert a result the evidence does not show.
> A unit the evidence/collection cannot exercise is **NOT-EVIDENCED / blocked**, never a silent PASS.
> Never invent an endpoint or field. Where the API collection is not yet wired, leave the
> **`[API SLOT — pending collection]`** marker rather than guessing.

---

## Step 0 — Frame


Read the story and name, in one line each:
1. **Actor** (product / OPM / Finance / approver / settlement party).
2. **Desired outcome** (what state the invoice/customer/points should reach).
3. **Money / tax / persisted-data impact?** If yes → this is a **blocking-grade** story; every
   money, VAT, state, or ZATCA rule must get a criterion (BA gate = `BLOCKED` otherwise).

## Step 1 — Classify the story against the knowledge base


Tick every capability the story touches (a story usually touches several):

| Capability | Trigger words in the story | Knowledge anchor |
|---|---|---|
| Onboarding | register product/service, Unified, Client Key | §1, BR-01/02 |
| Customer | customer create/update, CustomerNin, company vs individual | §2, BR-03/04 |
| Invoice create | create invoice, quote, subscription, transaction, amount, VAT | §2–3, BR-05…10 |
| Pricing model | dynamic / fixed / slice / per-unit / DPM / points | §3, BR-05…09, 33 |
| Sadad | Sadad number, upload, real-time/batch | §4, SadadUploadStatus |
| ZATCA | tax invoice, send, cleared/reported, QR | §4, BR-11…14 |
| Payment / paid | payment notification, out-of-system, bank transfer, OPM | §4, BR-15, 22 |
| Refund / credit memo | refund, credit note, reverse | BR-16…21 |
| Cancel / expiry | cancel, expired, MaxExtendPeriod, Job | BR-13, 23, 24 |
| Download | PDF, DownloadInvoice, financial claim | BR-30…32 |
| Notifications | expiry/payment notification, retry, outbox | §5.5, Type 4/5 |
| Points | points buy/consume/refund/expire | §3.4, §5.6, BR-26 |

## Step 2 — Pin the pricing model & integration surface


- If pricing is involved, decide **which model** (Dynamic / Fixed / Slice / PerUnit / Accumulative-GAP / **DPM**) and write the inputs the product sends and how the system computes (§3). This decides the **boundary** tests.
- List the **integrations** the flow crosses (§4). Each crossing becomes at least one **DB-layer** verification against the matching table in §5.

## Step 3 — Derive atomic testable units


For the story, generate units across **all** of these lenses (skip a lens only with a stated reason):

1. **Positive / happy path** — the main outcome reaches its state.
2. **Negative** — each violated rule from §6 that applies (e.g. BR-06 UnitPrice to non-dynamic service; BR-16 partial memo on New).
3. **Boundary** — Slice band edges, PerUnit high-volume rounding, due-date/expiry, date/time skew (BR-24), VAT-inclusive vs exclusive (BR-10).
4. **i18n EN/AR** — AR + EN service name/description present; AR description required for operations-invoice PDF; RTL rendering on screens/PDF.
5. **Tax** — VAT rate from config (BR-28), non-invoiceable → 0% and not sent to ZATCA (BR-27), ZATCA state reached (BR-11).
6. **Idempotency / UPSERT** — same `CustomerNin` twice = update not duplicate (BR-03); anti-re-billing stamp (BR-25).
7. **Notification / retry** — queued → mediation header processed → retried on failure; Type 4/5 (§5.5).

Each unit is **atomic** (one assertion) and **implementation-independent** in its objective.

## Step 4 — For each unit, assign WHERE to test (the 3 layers)


Every unit must state its layers. Use this decision guide:

- **API layer** — when the unit is about a request/response contract (create, push, refund, notify).
  Emit the **`[API SLOT — pending collection]`** marker: endpoint + method + key fields once the
  Postman collection is wired. Do **not** invent paths now.
- **DB layer** — the authoritative check for *did the system persist/compute/route correctly*.
  Name the **exact `{{TBL_*}}` table + column/condition** from §5 (e.g. "ZATCA reached? →
  `{{TBL_INVOICE_ZATCA}}`, status in {Reported,Cleared}"). This is where most verdicts are earned.
- **UI layer** — when the story has a screen outcome: **Invoices Management** (search by Sadad
  number / creation date / status; view; download; cancel; update payment status — role-gated),
  the **invoice/quote PDF** (DownloadInvoice), Unified service-entry screen, credit-memo approval link.

> Rule of thumb: API proves the *request was accepted*, **DB proves it was done right**, UI proves
> it is *visible to the user*. A money/tax/state story needs a DB-layer verification to be PASS-able.

## Step 5 — Write each unit into the output template (`method/output-template.md`) and set traceability


Map **story → knowledge rule (`BR-*`) → table/screen → criterion**. Unresolved choices that change
money, tax, permissions, or persisted data → mark the unit **BLOCKED (pending PO/collection)**, do
not guess.

---

## Self-check before handing a story's units to Planner/QA


- [ ] Every money/tax/state assertion has a **DB-layer** check naming an exact `{{TBL_*}}` table (§5).
- [ ] Every applicable `BR-*` from §6 produced at least one positive **and** one negative/boundary unit.
- [ ] i18n (AR+EN), idempotency/UPSERT, and notification/retry lenses each considered or explicitly waived.
- [ ] No invented endpoint/field — all API specifics are `[API SLOT — pending collection]`.
- [ ] Units that can't be exercised yet are labelled **NOT-EVIDENCED / BLOCKED**, not PASS.
