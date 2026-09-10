# Data-model map — table -> what it records -> test question

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> Fidelity labels: **FACT** (stated in source) / **INFERRED** / **GAP** (needs the API collection or PO).
> Never assert a result the evidence does not show; never invent an endpoint or field.

## 5. DATA MODEL MAP — table → what it records → test question it answers


> Keyed primarily by `INVOICE_NUMBER` / `INVOICE_ID`. `RELEATED_INVOICE_ID` (sic — misspelled in schema) links refunds/credit memos to the original.

### 5.1 Invoice core & financial

| Table | Records | Test question it answers |
|---|---|---|
| `{{TBL_INVOICE}}` | Invoice header: `INVOICE_ID`, `INVOICE_NUMBER`, status, amounts, VAT, `RELEATED_INVOICE_ID`, SadadUploadStatus, `CREATEDON` | "Was the invoice created, with what amount/VAT/status, and is it linked to an original?" |
| `{{TBL_INVOICE_HISTORY}}` | State-transition log per `INVOICE_NUMBER` | "What state transitions happened and in what order (New→Paid→Refund…)?" |
| `{{TBL_INVOICE_DETAILS}}` | Line items | "Do the line items (service, qty, price, VAT) match what was sent?" |
| `{{TBL_INVOICE_REFUND}}` | Refund rows (found via `RELEATED_INVOICE_ID` chain) | "Was a refund recorded and for what amount (pre-tax + 15%)?" |
| `{{TBL_INVOICE_DISCOUNT}}` | Discounts applied | "Was a discount applied and correctly reflected in totals?" |
| `{{TBL_INVOICE_QC}}` | INFERRED — per-invoice QC/quality-control or quote-control record (name only in SQL) | INFERRED: "Did the invoice pass internal QC gating?" — **confirm meaning with PO/collection.** |

### 5.2 Services / subscriptions / pricing

| Table | Records | Test question |
|---|---|---|
| `{{TBL_INVOICE_SERVICE_SUBSCRIPTION}}` | Subscription lines on the invoice | "Is the subscription (period, volume, value) correctly attached?" |
| `{{TBL_INVOICE_SERVICE_TRANSACTION}}` | Usage transactions attached to the invoice | "Which usage transactions were billed (and do they now carry this `invoiceId`)?" |
| `{{TBL_SERVICES}}` | Service master (`SERVICE_ID`, `SERVICE_EXTERNAL_ID`, `SERVICE_TYPE`, `SERVICE_HAS_VAT`, AR/EN name/desc, `SERVICE_ACTIVITY`) | "Is the service registered, active, with the right type and VAT%?" |
| `{{TBL_APPLICATIONS}}` | Product/application master (`APPLICATION_ID` = Product Code) | "Does the product exist with this code?" |
| `{{TBL_APPLICATIONS_SERVICES}}` | Product↔service binding (`APPLICATION_SERVICE_ID`) | "Is this service linked to this product?" |
| `{{TBL_SERVICE_PRICE_LIST}}` | Service↔price-list binding | "Which price list drives this service?" |
| `{{TBL_PRICE_LIST}}` | Price list (`PRICE_LIST_TYPE` 1=Range/2=Fixed, `PRICE_LIST_ACTIVITY`) | "Is it Range or Fixed, and is it active?" |
| `{{TBL_PRICE_LIST_DETAILS}}` | Detail rows: `COST`, `SLICE_START_VOLUME`, `IS_QUANTITATIVE` (0=Slice/1=CalculatePerUnit/2=Accumulative), `SLICE_START_DATE`, `PRICE_LIST_DETAIL_ACTIVITY` | "Which band/price applies to this Volume, and is the right (active) detail used?" |

### 5.3 Sadad / payment

| Table | Records | Test question |
|---|---|---|
| `{{TBL_INVOICE_SADAD}}` | Sadad number + upload info per invoice | "Does the invoice have a Sadad number and what upload status?" |
| `{{TBL_SADAD_TRACK_TRANSACTION}}` | Sadad upload transaction header (`TRANSACTION_ID`) | "Did the upload transaction run, when, with what result?" |
| `{{TBL_SADAD_TRACK_TRANSACTION_DETAILS}}` | Upload detail; `EFFECTED_NUMBER` = `INVOICE_ID` | "What request/response/message for this invoice's Sadad push?" |
| `{{TBL_PAYMENT}}` | Payment records per invoice | "Was a payment recorded against the invoice?" |
| `{{TBL_PAYMENT_SADAD}}` | Sadad-specific payment detail (`PAYMENT_ID`) | "What are the Sadad payment specifics?" |

### 5.4 Tax

| Table | Records | Test question |
|---|---|---|
| `{{TBL_INVOICE_ZATCA}}` | ZATCA send records (`SENDING_DATE`, status, response) | **"Did the invoice reach ZATCA and get a response (Reported/Cleared/Error)?"** |

### 5.5 Notifications / mediation / outbox

> **Type codes: `5` = Expired, `4` = Payment.** For every notification, check it was **processed**, and if it failed, check the **retry**.

| Table | Records | Test question |
|---|---|---|
| `{{TBL_EXPIRY_NOTIFICATION_QUEUE}}` | Expiry notifications queued (`EXPIRY_NOTIFICATION_QUEUE_INVOICE_NUMBER`) | "Is an expiry notification queued for this invoice?" |
| `{{TBL_MEDIATION_SEND_EXPIRY_NOTIFICATION}}` | Expiry-notification send rows; FK `MEDIATION_SEND_HEADER_ID` | "Was the expiry notification dispatched via mediation?" |
| `{{TBL_PAYMENT_NOTIFICATION_QUEUE}}` | Payment notifications queued (`PAYMENT_NOTIFICATION_QUEUE_Invoice_Number`) | "Is a payment notification queued?" |
| `{{TBL_MEDIATION_SEND_PAYMENT_NOTIFICATION}}` | Payment-notification send rows | "Was the payment notification dispatched?" |
| `{{TBL_MEDIATION_SEND_HEADERS}}` | Mediation send headers (`HEADER_ID`, Type 4/5, processed flag, retry) | "Was it processed? Did it retry on failure? Which type (4 payment / 5 expiry)?" |
| `TBL_OUTBOX_MESSAGE` | Inbound notifications from the payment server (`INVOICE_NUMBER`, `CONTENT`, `CREATED_ON`) | "Did the payment server's notification land in the outbox for this invoice?" (search by `INVOICE_NUMBER` **or** `CONTENT LIKE '%…%'`). |

> **Known header-query bug in the source (do not copy):** the *payment* header query joins back through
> `{{TBL_MEDIATION_SEND_EXPIRY_NOTIFICATION}}` instead of the payment-notification table — so it returns
> expiry headers, not payment headers. Flag this; the correct join is via
> `{{TBL_MEDIATION_SEND_PAYMENT_NOTIFICATION}}`. **INFERRED correction.**

### 5.6 Points

| Table | Records | Test question |
|---|---|---|
| `{{TBL_POINTS_RECIEVED_HEADERS}}` | Points message headers (`HEADER_ID`, `MSG_FROM`=Product Code, `REQUEST_ID`, `CREATED_ON`) | "Did the points message arrive from this product?" |
| `{{TBL_POINTS_RECIEVED_CONSUMPTION}}` | Points consumed (`ConsumedPoints`, `ConsumedAmount`, `MOINumber`=customer, `ServiceId`, `TransactionDate`) | "How many points/amount consumed per customer/service/date?" |
| `{{TBL_POINTS_RECIEVED_REFUND}}` | Points refunded (`RefundPoints`, `RefundAmount`) | "How many points/amount refunded?" |
| `{{TBL_POINTS_RECIEVED_EXPIERED}}` | Points expired (`ExpiredPoints`, `ExpiredAmount`) | "How many points/amount expired?" |

---
