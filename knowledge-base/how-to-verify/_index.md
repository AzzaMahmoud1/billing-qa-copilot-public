# How-to-verify — capability → recipe (the routing table)

> The agent's lookup path. Match the story's capability (from method Step 1) to a recipe, then follow it.
> Each recipe names the **API family** (`endpoints.md`), the **tables** (`data-model.md`), the **screen**
> (`screens.md`), a **SQL side-effect probe**, and the **edge cases** (`business-rules.md`).

| Capability (trigger words) | Recipe | Primary tables |
|---|---|---|
| create invoice, quote, subscription, transaction, amount, VAT, pricing | [`create-invoice.md`](create-invoice.md) | `{{TBL_INVOICE}}`, `{{TBL_INVOICE_DETAILS}}` |
| tax invoice, ZATCA, reported, cleared, QR | [`zatca-reporting.md`](zatca-reporting.md) | `{{TBL_INVOICE_ZATCA}}` |
| Sadad number, upload, real-time/batch | [`sadad-upload.md`](sadad-upload.md) | `{{TBL_INVOICE_SADAD}}`, `{{TBL_SADAD_TRACK_TRANSACTION}}*` |
| refund, credit note/memo, reverse, cancel | [`refund-and-credit-memo.md`](refund-and-credit-memo.md) | `{{TBL_INVOICE_REFUND}}`, `{{TBL_INVOICE}}` (`RELEATED_INVOICE_ID`) |
| points buy/consume/refund/expire, loyalty | [`points-redemption.md`](points-redemption.md) | `{{TBL_POINTS_RECIEVED_}}*` |
| on-behalf, proxy, third-party, multi-party billing | [`proxy-3rd-party-billing.md`](proxy-3rd-party-billing.md) | `{{TBL_INVOICE}}` + Company Profile |
| payment notification, paid, bank transfer, OPM | (use `create-invoice.md` §Paid + `data-model.md` §5.5) | `{{TBL_PAYMENT}}*`, notification/outbox tables |
| expiry, MaxExtendPeriod, Job | (use `data-model.md` §5.5 + BR-23/24) | `{{TBL_EXPIRY_NOTIFICATION_QUEUE}}`, mediation |

**If no recipe matches:** fall back to the method (`method/story-analysis-method.md`) and the data-model
map directly. If the HO is silent on the capability, say **"not covered"** and point at
`coverage-gaps.md` — do not invent.
