# Changelog

The knowledge base is a **living** asset. Record every domain fact added/changed (with its fidelity
label) and every sanitization/structure change.

## [Unreleased] — 2026-09-15 — Rabet catalog: full execution, eligibility mechanism, source verification
### Changed / corrected
- **Endpoint correction:** services live at `GET /api/service/list/{packageId?}` (route param) per OBS-11391 — the earlier `/service/details` notes were the wrong endpoint. `/service/list` returns `priceListItems[]` (retracts the "no priceListItems" finding).
- **Eligibility mechanism found:** `EnableRabetInquiry` is a **row** in `{{TBL_SETTING}}` (`SET_CODE`/`SET_VALUE 1|0`/`SET_REFERENCE`=product), and it is **enforced** — product `992` (flag=1, 0 packages) → `1193` (eligible-but-empty) vs `199` (no flag) → `1005` (not eligible). Corrects the old "flag not in BPQA / eligibility is 3scale-only". Matches the use-case Pre-Condition. `999` works legacy (no flag row).
### Verified against source (Confluence BIL/264631311 + 3 use-case docs, ticket comments)
- Exception flows are explicit: **E2→Empty Package List, E3→Empty Services List, E4→Empty Price List Items** (empty arrays, not errors). API returns errors (`1193`/`1191`/omission) → **confirmed defects**.
- Field-name mismatch spans Product **and** Package/Service (`Code/EnglishName` vs `packageCode/serviceEnglishName`); `isQuantitative`(0/1) vs API `amountMultiplied`(0/1/2); `PriceListCreationDate` presence unverified.
- OBS-8527 / OBS-11391: 0 comments, 0 attachments.
### Jira (authorized writes)
- 12 execution comments on OBS-11681…11692; bugs **OBS-11693** (E3), **OBS-11694** (input msgs/E1), **OBS-11695** (E4), **OBS-11696** (field names), **OBS-11697** (E2/TC-008) — each Blocks plan OBS-11680 + links its case. Correction comments on TC-005/TC-008/E4/field-names.
### Gaps still pending live execution
- `PriceListCreationDate`, `isQuantitative`/`amountMultiplied` value-set, VAT inheritance, "items limit on API", dynamic-services→E4, TC-005 flag-toggle, TC-006 non-Rabet caller.

## [Unreleased] — 2026-09-14 — Rabet Get Product Catalog Details
### Added
- `knowledge-base/rabet-product-catalog.md` — the Rabet catalog (OBS-UC086A/B/C) is three endpoints
  (`product/list`, `package/list`, `service/details`), Rabet-only, keyed by `Product_Code`; inputs, response
  shape, known product codes/services, credentials location (sheet, secret not copied), and QA findings.
### QA findings (2026-09-14)
- **`EnableRabetInquiry` flag is not in the BPQA schema yet** (no `%RABET%`/`%INQUIR%` column) → no eligible
  product → happy-path + empty-array flows BLOCKED. Flag stories OBS-8612/ELMX-7741 Open; API story OBS-8527 Development.
- Rabet app-Id `{{rabetAppId}}` authenticates at the gateway; but normal billing creds also return data → **"only Rabet"
  restriction not yet enforced**.
- Error-message mismatches vs spec: "ProductCode is missing in request Header" (want "ProductCode is required");
  "The Application Not Exists" (want "ProductCode does not exist").
- Test plan **OBS-11680** + cases **OBS-11681…OBS-11692** created for this use case.
### QA findings (2026-09-14, cont.) — TC-011/012 executed
- Created empty package `QA-RABET-EMPTY-011` (PACKAGE_ID 989) on product 999 in BPQA (authorized DB write) to unblock TC-011.
- **TC-011 NOT-EVIDENCED / not exercisable:** the Services endpoint has **no package filter** — `Package_ID` as header/query is ignored (returns all services), and `/service/details/{code}` reads the path segment as a *service* code (`1190 "Service does not exist under this product."`). `GetServiceByPackageV3` in the collection is an unconfigured stub (URL = `/package/list`). → **spec-vs-impl gap** vs OBS-UC086C ("Package_ID optional, null = all").
- **TC-012 executed:** empty price list (service `NEWFARESSSS`, 0 price rows) returns an **error** `1007 "The Price List Not Exists - {0}"`, not an empty collection; priced control `Test_PD-UMRH-009` returns `priceListId 40016`. Disposition (error vs empty) pending OBS-11692 AC. Also: the `{0}` placeholder in message 1007 is **unresolved** (formatting defect).
- `knowledge-base/rabet-product-catalog.md` updated with endpoint mechanics (service/details, service/{code}, /pricelist) and findings F4–F6.

## [Unreleased] — 2026-09-13 — API collection wired
### Added
- `api-collection/collection.json` — the real Postman export **"Elm Billing System (EBS)"**
  (v2.1, 39 folders / 132 requests), **sanitized before landing** via `../scrub_api_collection.py`:
  1009 redactions — auth blocks, `Authorization`/`X-API-Key` headers, 58 saved example responses,
  an X.509 WS-Security cert + SOAP signatures/digests, and embedded PII (customer NINs, account/bill/
  VAT numbers, hex hashes, emails). No secrets committed; real tokens/URLs stay in gitignored
  `environment.json` per `api-collection/README.md`.

### Notes
- Folder names (`Json`, `SADAD Endpoints - SOAP - Callbacks`, …) do **not** match capability ids, so
  the agent uses **endpoint-family** matching (README fallback) rather than folder→capability keys.
- **Before any public push:** regenerate the public build (`python tools/sanitize.py
  ../billing-qa-copilot-public`) and confirm the **zero-hit leak gate** passes with the new file.

### Public exposure decision
- The **real** `collection.json` is **private-only** — added to `tools/sanitize.py` exclude list, so it
  never enters the public build. The public repo instead ships **`collection.example.json`**, a synthetic
  collection (placeholders / `{{vars}}` / `<SAMPLE_INVOICE_NO>`, WS-Security omitted) that demonstrates
  the ingest conventions without exposing the live EBS/SADAD API surface.

## [0.1.0] — 2026-09-10 — Initial seed from the HO handover
### Added
- Operating contract (`CLAUDE.md`) + Claude Code agent definition (`.claude/agents/billing-qa-copilot.md`).
- Knowledge base seeded from the HO handover: `glossary`, `invoice-lifecycle`, `pricing-models`,
  `integrations`, `data-model` (~30 `{{TBL_*}}` tables mapped), `business-rules` (BR-01..BR-34),
  `endpoints` (families; concrete paths are `[API SLOT — pending collection]`), `screens`,
  `coverage-gaps`, and six `how-to-verify/` recipes.
- Reusable `method/` (story-analysis procedure, test-design checklist, output template).
- `api-collection/` drop-in slot (README + placeholder collection + example environment).
- Worked `examples/example-create-invoice.md`.
- Dual-repo tooling: `tools/sanitize.py` (public build + zero-hit leak gate), `SANITIZATION.md`.

### Notes / carried-forward gaps
- All endpoint paths/schemas pending the forthcoming Postman collection.
- `{{TBL_INVOICE_QC}}` meaning INFERRED; `IS_QUANTITATIVE=2` (Accumulative) rule unexplained; DPM tables
  not enumerated; Etimad status values unknown. See `knowledge-base/coverage-gaps.md`.
- Flagged a **bug in the source SQL**: the payment-notification header query joins through the *expiry*
  notification table; corrected join noted in `data-model.md` §5.5 (confirm before reuse).
