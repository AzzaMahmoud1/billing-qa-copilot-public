# Test-Design Checklist (coverage lenses)

> Applied in Step 3 of `story-analysis-method.md`. For a covered capability, generate units across
> **every** lens below; skip a lens only with a stated reason. Each lens links back to the knowledge
> base so the edge cases are emitted automatically, not from memory.

## The lenses

| # | Lens | What to generate | Anchor in `knowledge-base/` |
|---|---|---|---|
| 1 | **Positive / happy path** | The main outcome reaches its target state. | `invoice-lifecycle.md` |
| 2 | **Negative** | One unit per violated rule that applies to the story. | `business-rules.md` (`BR-*`) |
| 3 | **Boundary** | Slice band edges; PerUnit high-volume rounding; due-date/expiry; date/time skew (BR-24); VAT-inclusive vs exclusive (BR-10). | `pricing-models.md`, `business-rules.md` |
| 4 | **Contract / schema** | Request/response shape; required vs optional fields; error-code conformance (exact strings, EN/AR). | `endpoints.md` + the API collection |
| 5 | **i18n EN/AR** | AR + EN service name/description present; AR description required for the operations-invoice PDF; RTL rendering on screens/PDF. | `glossary.md`, `screens.md` |
| 6 | **Tax** | VAT rate comes from config (BR-28); non-invoiceable → 0% and not sent to ZATCA (BR-27); ZATCA state actually reached (BR-11). | `integrations.md`, `business-rules.md` |
| 7 | **Idempotency / UPSERT** | Same `CustomerNin` twice updates, not duplicates (BR-03); anti-re-billing stamp on usage transactions (BR-25). | `business-rules.md` |
| 8 | **Notification / retry** | Queued → mediation header processed → retried on failure; Type **4 = payment**, **5 = expiry**. | `data-model.md` §5.5 |
| 9 | **AuthN / AuthZ** | Role-gated screen actions (view/cancel/update-payment on Invoices Management); Client Key / authenticated billing role. | `screens.md`, `glossary.md` |

## Domain edge cases to auto-emit (the "surprise dividend")

When a story touches these areas, surface the compliance edge case **without being asked**:

- **ZATCA:** `New → Reported → Cleared/Error`; never infer `Cleared` from a happy-path screen; a tax
  invoice **cannot be cancelled** after send (BR-13) → use credit note / refund.
- **Sadad:** `NotUploaded` stuck vs `DoNotUpload` (system decided never to push); Real-Time (manual
  API) vs Batch (auto, ≤24h).
- **Refund / credit memo:** refund amount entered **pre-tax**, system adds 15% (BR-21); credit memo on
  a `New` invoice must equal full due (BR-16); partial only **after** payment (BR-17); needs ZATCA
  `Cleared`/`Reported` (BR-18).
- **Points:** **no automatic balance check** (BR-26) — call it out, do not invent a validation.
- **Pricing:** Slice applies a flat band price (no multiply, BR-08); PerUnit multiplies (BR-09);
  VAT-inclusive vs exclusive changes the total.

## Self-check before handing units on

- [ ] Every money/tax/state assertion names an exact `{{TBL_*}}` table (DB-layer check).
- [ ] Every applicable `BR-*` produced ≥1 positive **and** ≥1 negative/boundary unit.
- [ ] i18n, idempotency/UPSERT, notification/retry each considered or explicitly waived.
- [ ] No invented endpoint/field — API specifics are `[API SLOT — pending collection]`.
- [ ] Unexercisable units are labelled **NOT-EVIDENCED / BLOCKED**, not PASS.
