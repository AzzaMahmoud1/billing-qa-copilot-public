# Pricing models (Dynamic / Fixed / Slice / PerUnit / DPM / Points)

> Part of the **Billing QA-Copilot** knowledge base. Source: HO handover doc.
> Fidelity labels: **FACT** (stated in source) / **INFERRED** / **GAP** (needs the API collection or PO).
> Never assert a result the evidence does not show; never invent an endpoint or field.

## 3. The pricing models


> The source distinguishes **two layers** that both get called "dynamic" — keep them apart:
> (a) the four **price-list calculation models** used in *normal* billing, and
> (b) the **Dynamic Pricing Module (DPM)**, a separate path. Both are documented here.

### 3.1 The four normal-billing pricing models

| Model (narrative term) | Product sends | System computes | Schema mapping |
|---|---|---|---|
| **Dynamic pricing** (consistency-checked) | `Volume` + `UnitPrice` + `Amount` | **Validates** that Amount is consistent with Volume × UnitPrice before issuing. May require Finance approval for some products. | INFERRED: not a single `IS_QUANTITATIVE` flag; it is the product sending all three values. |
| **Defined / Fixed** | `Volume` only | Price is preconfigured; system compares volume to the configured price and computes amount. | `PRICE_LIST_TYPE = 2` (Fixed). |
| **Slice** | `Volume` only | Each quantity **band** has one flat price; system finds the band containing Volume and applies its flat price — **no multiplication** by quantity. | `PRICE_LIST_TYPE = 1` (Range) + `IS_QUANTITATIVE = 0` (Slice); band floor = `SLICE_START_VOLUME`. |
| **PerUnit** | `Volume` only | Each band carries a per-unit price; amount = **Volume × per-unit price** of the band. | `PRICE_LIST_TYPE = 1` (Range) + `IS_QUANTITATIVE = 1` (CalculatePerUnit). |
| **Accumulative** | `Volume` | **GAP** — `IS_QUANTITATIVE = 2` appears in SQL but the narrative does not explain it. Flag for PO/collection. | `IS_QUANTITATIVE = 2`. |

**Worked numeric examples (from source):**

- **Dynamic / general VAT:** base **1000** → VAT 15% = **150** → total **1150**.
- **Slice:** qty **1–4** → flat **5 SAR**; qty **5+** → a different flat price. System applies the band's flat price, does not multiply. (Pre-tax.)
- **PerUnit:** qty 1 @ 5 = **5**; qty 2 = **10**; qty **11** in a band priced **15/unit** = **11 × 15 = 165** pre-tax.
- **Defined/Fixed:** INFERRED — amount = configured unit price × Volume against the fixed price row (example not given verbatim; confirm against collection).

**VAT-inclusive vs exclusive (important rounding rule):** the system needs to know whether the product-sent price is **VAT-inclusive or exclusive**. If exclusive, it adds 15%; if inclusive, it treats it as the final price. Decimal prices × large volumes can produce large rounding deltas — a price that looks near-integer at qty 1 can drift at millions of units.

**Most-likely defects to test per model:**
- **Dynamic:** (1) Amount inconsistent with Volume × UnitPrice but still accepted; (2) `UnitPrice` sent to a service **not** configured as dynamic (known error); (3) missing Finance-approval gate where required.
- **Defined/Fixed:** (1) subscription/volume not matching the configured price list (known error — "حجم الاشتراك لا يطابق قائمة الأسعار"); (2) amount computed against an inactive/old price-list detail.
- **Slice:** (1) boundary band misassignment (qty exactly at `SLICE_START_VOLUME`); (2) system multiplying by quantity instead of applying the flat band price.
- **PerUnit:** (1) wrong band selected at a boundary volume; (2) rounding drift at high volume; (3) VAT-inclusive/exclusive flag misread.

### 3.2 Dynamic Pricing Module (DPM) — separate path

- Product sends **Service ID + Volume + Amount**; system does **not** recompute from the price list — it takes the amount, **adds VAT**, issues a **subscription invoice directly**.
- Multiple services per subscription request (service id, description, start/end date, volume, pre-tax amount); system sums them, then computes total + VAT.
- **Uses different settings and tables** from normal billing (exact tables = GAP).
- Likely defects: VAT added twice / not added; service amounts summed incorrectly; DPM invoice wrongly routed through the quote→tax-invoice staging instead of issuing directly.

### 3.3 Usage transactions & postpaid operations invoice

- Postpaid model: usage transactions are recorded (`customerNin`, `contractId`, `contractItemId`, `serviceId`, Volume, pre-tax amount, transaction date), then an **operations invoice** is generated for a period (month/quarter).
- An **edit/exclude** endpoint can correct or exclude mis-recorded transactions **before** invoicing (was not used in prod at time of recording — GAP on availability).
- **Anti-re-billing rule:** once an invoice is created, its `invoiceId` is stamped on the linked usage transactions; re-billing the same period **excludes** transactions already carrying an `invoiceId`; un-billed ones remain includable.
- Arabic service description (`ServiceArDescription`) must exist — the operations-invoice PDF uses it.

### 3.4 Points

- Points are sold as a service with a price list (e.g. point = 2.50 SAR pre-tax + VAT); **buying** points creates a normal invoice. Billing's responsibility = proof of purchase, **not** tracking later usage.
- Post-purchase flows: **consumption, refund, expiry** — each a separate endpoint/record set.
- **No automatic balance check:** the system records what the product sends; if the product sends more than purchased, it is stored as-is with no comparison. Reconciliation is manual (monthly/quarterly email review). The future **Wallet** is intended to give a central, verifiable balance.

---
