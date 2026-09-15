# Rabet "Get Product Catalog Details" (OBS-UC086) — Concerns against the documents

> QA concerns raised as postable comments. Each is a **discrepancy between the built API and a
> requirement document** (use case OBS-UC086 A/B/C, the Restrictions section, the error-code/AC spec,
> and the eligibility-flag stories ELMX-7741 / OBS-8612). Verified live on 2026-09-14 as the Rabet
> app (app-Id {{rabetAppId}}) against product 999 in QA. No secrets included. Severity is QA's opinion;
> PASS/CONCERN/FAIL where AC text is needed is marked "confirm AC".

---

## C1 — No per-package services retrieval (contradicts OBS-UC086C)
**Against:** OBS-UC086C "Retrieve Services — Package_ID optional (null = all)".
**Expected (doc):** The Services API can be filtered by package; passing a Package_ID returns that
package's services, null returns all.
**Observed:** No package filter exists. `GET /billing/v3/api/service/details/` returns **all** services
for the ProductCode. `Package_ID` as an HTTP **header** is ignored (full list); as a **path segment**
(`/service/details/{X}`) it is read as a *service* code and returns `1190 "Service does not exist under
this product."`; as a **query param** (`?Package_ID=`) it is ignored. The collection request
`GetServiceByPackageV3` is an unconfigured stub whose URL is `/package/list`.
**Impact:** The per-package services use case (incl. empty-services / TC-011) is **not testable** and,
as built, not deliverable per spec.
**Severity:** High (missing spec'd capability). Confirm against API story OBS-8527 build state.

## C2 — Empty price list returns an error, not an empty collection (TC-012 / OBS-11692)
**Against:** OBS-UC086 empty-collection AC (TC-012, case OBS-11692).
**Expected (doc/oracle):** A service with no price list returns an empty price list / no items.
**Observed:** `GET /billing/v3/api/service/NEWFARESSSS/pricelist` (service on 999 with 0 price rows)
returns `header.status:1, errorsCount:1, body.data:null`, error `key 1007 "The Price List Not Exists - {0}"`.
Priced control `Test_PD-UMRH-009` returns `data:[{priceListId:40016,…}]`.
**Impact:** If the AC expects an empty collection, returning an error is a non-conformance; consumers
must special-case an error for a legitimate "no price yet" state.
**Severity:** CONCERN → confirm AC (empty-collection vs error). Likely FAIL against an empty-list oracle.

## C3 — Unresolved "{0}" placeholder in error message 1007
**Against:** Error-message/string spec (message formatting).
**Expected (doc):** A complete, human-readable message.
**Observed:** Error 1007 message is literally `"The Price List Not Exists - {0}"` — the `{0}` token is
not substituted.
**Impact:** Broken/placeholder text surfaced to the caller.
**Severity:** Low (cosmetic/formatting), but customer-visible.

## C4 — Product field names differ from spec (OBS-UC086A)
**Against:** OBS-UC086A response schema.
**Expected (doc):** `productCode / productEnglishName / productAR_Description / productENG_Description`.
**Observed:** `applicationCode / applicationNameEn / applicationNameAr / applicationDescEn / applicationDescAr`.
**Impact:** Contract mismatch; consumer mapping breaks against the documented schema.
**Severity:** Medium (contract).

## C5 — Services response shape differs from spec (OBS-UC086C)
**Against:** OBS-UC086C response schema.
**Expected (doc):** numeric `Type`, `priceListID`/`priceListType`, and a `priceListItems[]` array on each service.
**Observed:** `/service/details` returns flat `serviceType` (string) + `rate` + `pricingModel`; **no
`priceListItems[]`** on the service. (Price data is only reachable via the separate
`/service/{code}/pricelist` endpoint.)
**Impact:** Consumers expecting embedded price items per the spec get none; different integration shape.
**Severity:** Medium (contract).

## C6 — Error-message wording differs from spec
**Against:** Error-code/AC spec (same pattern noted in OBS-9911).
**Expected (doc) → Observed:**
- "ProductCode is required" → **"ProductCode is missing in request Header"** (5001)
- "ProductCode does not exist" → **"The Application Not Exists"** (1005)
- (format) → **"ProductCode is invalid format"** (5002)
**Impact:** Message contract mismatch; automated consumers keying on wording will break.
**Severity:** Low–Medium (contract/UX).

## C7 — "Only Rabet" access is not enforced at the application level (Restriction #1)
**Against:** Restrictions §1 — "Only Rabet should be able to access Get Product Catalog API… Rabet
Client Key will be hardcoded and validated."
**Expected (doc):** Only the Rabet consumer can call the catalog APIs.
**Observed:**
- ClientKey **is** validated — a wrong ClientKey returns `5008 "Invalid ClientKey"` (verified 2026-09-14). ✅ that clause holds.
- **But** the validated key is the generic `Billing_Client_Key`, not a Rabet-specific key, so it gates
  *billing* callers, not *Rabet-only*.
- Prior run: **normal (non-Rabet) billing credentials also returned catalog data** → the app-level
  "only Rabet" gate is **not enforced**.
**Impact:** Any billing consumer with the shared client key + a valid 3scale app may read the catalog —
the core access restriction is not met. **Security-relevant.**
**Severity:** High. Definitive re-test needs a valid non-Rabet app credential (not in the KB; would
require another environment/app — flag to PO). Confirm intended enforcement point (3scale subscription).

## C8 — "Allow Rabet inquiry" eligibility mechanism not implemented as documented (Restriction #2)
**Against:** Restrictions §2 + ELMX-7741 (enable "Allow Rabet inquiry" on onboarding) + OBS-8612
(manage it in New Billing Portal product details).
**Expected (doc):** Eligibility is driven by an "Allow Rabet inquiry" flag set at onboarding and
manageable in the portal.
**Observed:** No such flag exists in BPQA (`EnableRabetInquiry` / `%RABET%`/`%INQUIR%` columns = 0).
Eligibility is currently only the 3scale subscription (only product 999 eligible), external to billing.
ELMX-7741 / OBS-8612 still Open.
**Impact:** The documented, manageable eligibility control does not yet exist; eligibility is opaque and
not portal-manageable.
**Severity:** Medium–High (feature not built as specified). Track with ELMX-7741 / OBS-8612.

## C9 — In-scope product mapping unverified (Restriction #2)
**Against:** Restrictions §2 — scope limited to "Absher B2B Manual Contracts".
**Expected (doc):** Rabet inquires only in-scope products; the in-scope product is Absher B2B Manual Contracts.
**Observed:** The only eligible test product is `999`, a QA scratch product (contains test packages
like `NEWWWWW`, `new`, `TEST_PKG-PD-UMRH-009`). It is **unconfirmed** that 999 = "Absher B2B Manual
Contracts"; no production in-scope product has been evidenced.
**Impact:** Cannot confirm the restriction is exercised against the real in-scope product.
**Severity:** Medium (test-data / traceability). QA can verify 999's product name in BPQA on request.

## C10 — Ineligible vs non-existent product indistinguishable (E1)
**Against:** Error handling for eligibility.
**Observed:** ineligible-existing `199` and non-existent `234` both return `1005 "The Application Not
Exists"`.
**Impact:** May be intended (per E1), but flagged: a caller cannot distinguish "exists but not allowed"
from "does not exist."
**Severity:** Low (confirm intended).

## C11 — TC-008 (empty packages) not evidenced — coverage gap
**Against:** Empty-collection AC (TC-008).
**Observed:** Needs an eligible product with no packages; only 999 is eligible and it has packages, so
the empty-packages case is not sourceable.
**Impact:** Coverage gap for the empty-packages scenario.
**Severity:** Low (coverage). Same root cause as the opaque eligibility set (C8).

---

### Verdict roll-up
- **Enforced / OK:** ClientKey validation (C7, partial); eligibility *effect* (999 in / 199,234 out).
- **Confirmed discrepancies vs docs:** C1, C3, C4, C5, C6.
- **Confirm AC / PO:** C2, C9, C10.
- **Not-yet-built vs docs:** C7 (app-level only-Rabet), C8.
- **Coverage gaps:** C11 (and C1 blocks TC-011).

_Evidence retained in `knowledge-base/rabet-product-catalog.md` (endpoint mechanics + F4–F6) and this
session. Jira (jira-elm) was unreachable this session, so AC text for OBS-11681…11692 was not re-read;
items marked "confirm AC" need that check._
