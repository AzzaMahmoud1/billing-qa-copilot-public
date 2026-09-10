# Integration points (Unified / Sadad / ZATCA / Finance / Etimad)

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> Fidelity labels: **FACT** (stated in source) / **INFERRED** / **GAP** (needs the API collection or PO).
> Never assert a result the evidence does not show; never invent an endpoint or field.

## 4. Integration points


| Integration | Direction | Purpose | What can go wrong (test focus) |
|---|---|---|---|
| **Unified (Portal)** | Product → Billing | Onboard product (`Product Code` + `Client Key`) and services (`Service ID`, AR/EN name, type, VAT%, pricing model, non-invoiceable flag); review → approval → QC. | Product/Service id present in one system only → billing cycle blocked; service not pre-registered → invoice request fails; mismatched Product Code across systems. |
| **Sadad** | Billing ↔ Sadad | Invoice gets Sadad Number; product queries + pays. Real-Time (manual API push) vs Batch (auto, ≤24h). | `NotUploaded` stuck; `DoNotUpload` set wrongly; Sadad number missing so product can't pay; upload transaction error/response not tracked. |
| **ZATCA** | Billing → ZATCA → Billing | Tax invoices sent (XML with QR, seller/buyer, address, tax number, tax detail); status `New→Reported→Cleared/Error`; requires Sequence Number. | Sent without seq number; stuck waiting for response; `Error` not followed up; tax invoice cancelled directly (forbidden); config says "send" but cache not refreshed. |
| **EBS / Finance + OPM** | Out-of-band | Out-of-system payments (cash/cheque/bank transfer): Finance confirms receipt → OPM manually marks invoice **Paid** with reference number + payment method. System does **not** auto-verify the bank. | Invoice marked Paid without Finance confirmation; missing reference number; wrong invoice matched. |
| **Settlement / Payment-notification party** | → Billing | Sends **Payment Notification** (operation number, bank-transfer number, bank code, channel, method, time, amount, RRN, card digits when needed) → system flips invoice to **Paid** and notifies product. | Notification not processed; invoice not flipped to Paid; retry not firing (Type 4). |
| **SAP** | Product → Billing (stored) | SAP customer number stored on the customer record when available. | Not present; not displayed where expected. |
| **Etimad (اعتماد)** | Billing ↔ Govt | Financial Claim raised for govt approval; becomes tax invoice after approval + payment. Finance refused giving products delete rights on govt transactions (MoH) — processing must go through Finance, not direct delete. | Claim downloaded as invoice before approval; delete attempted instead of Finance workflow. |
| **Redis cache** | Internal | Document download caching. | **Known bug:** download returns the **old** cached copy; temporary workaround was changing the description to force a new copy. |
| **Windows services** | Internal | `Invoice Generator`, `ZatcaIntegration` process the send cycle; request/response files saved on server (can be decoded to review XML). | Service down / file path / message errors; server **time skew (~3h)** caused invalid future-date errors (infra ticket opened). |

---
