# Rabet — Get Product Catalog Details (OBS-UC086)

> QA-discovered facts for the Rabet "Get Product Catalog Details" use case. Source: Confluence
> BIL page 264631311 (+ use-case docx OBS-UC086A/B/C, xlsx sheets), Jira OBS-8527/8437/8612/ELMX-7741,
> and live QA testing on 2026-09-14. Fidelity: FACT unless marked INFERRED/GAP.

## What it is
The "Product Catalog" is **three independent endpoints** (one use case each), Rabet-only, keyed by
`Product_Code`. They map onto existing product/package/service endpoints:

| Use case | API | Endpoint | New in scope |
|---|---|---|---|
| OBS-UC086A Retrieve Products | Retrieve Products | `GET /billing/v3/api/product/list` | adds product descriptions |
| OBS-UC086B Retrieve Packages | Retrieve Packages | `GET /billing/v3/api/package/list` | output already matches |
| OBS-UC086C Retrieve Services | Retrieve Services | `GET /billing/v3/api/service/details/` | `Package_ID` optional (null = all) |

**Inputs (headers):** `app-Id`, `app-key`, `Product_Code` (String 3-9, required); `Package_ID` optional on Services.
**Envelope:** `{header:{status,errorsCount,...}, body:{data, errors:[{key,field,message}]}}`.

## "Rabet" — two roles (2026-09-14)
- **Consumer/caller:** the 3scale application (`app-Id {{rabetAppId}}`) that invokes the catalog APIs (the use-case Primary Actor / marketplace).
- **Also a billing product:** `{{TBL_APPLICATIONS}}` has a row **`APPLICATION_ID 77`, EN `Rabet`, AR `رابط`** — the marketplace is itself onboarded as a product. (Code `77` is len 2, so it fails the API's 3-9 format rule → not usable as a catalog `Product_Code`.)

## Access, eligibility & test data
- Rabet-only access is by validated Rabet application credentials. The Rabet **app-Id/app-key** are in the page's
  xlsx (page 264631311). **Do not copy the app-key here** (secret; keep it in the sheet). Rabet **app-Id `{{rabetAppId}}`** authenticates.
- **Eligibility IS applied:** `ProductCode 999` (eligible) returns data; `199`/`234` return "The Application Not Exists"
  (existing-but-ineligible and non-existent both map to the same error, per E1). Eligibility column not found by name
  (`%RABET%`/`%INQUIR%` = 0 in BPQA) — stored under another name or 999 pre-configured. Flag-management UI (OBS-8612/ELMX-7741) still Open.
- **CORRECTION #3 (2026-09-14 #3) - `EnableRabetInquiry` IS the eligibility flag, and it IS enforced.** Earlier passes (and the old KB) said the flag "does not exist in BPQA" and eligibility is "only 3scale" - **both wrong.** The flag lives in **`{{TBL_SETTING}}`** as a **row**: `SET_CODE='EnableRabetInquiry'`, `SET_VALUE` (1=Enabled/0=Disabled), keyed by `SET_REFERENCE`=product Application ID (`SET_DESCRIPTION`: "1: Enabled, 0: Disabled - Enable Rabet Inquiry"). The prior `%RABET%/%INQUIR%` search missed it because it scanned **column names**, not `SET_CODE` **values**. **Proof it is enforced:** product `992` (EN `test`, `EnableRabetInquiry=1`, **0 packages**) -> Rabet `package/list` returns `1193 "No packages found for the given product."` (i.e. **eligible**, just empty), whereas `199` (no flag) -> `1005 "The Application Not Exists"` (**not eligible**). `1005` = not eligible; `1193`/`1191` = eligible-but-empty. Enabled products seen: `337,338,339,346,347,348,992,999000043` (+ more). `999` works but has **no** EnableRabetInquiry row -> legacy/separately-enabled.
- **So a product CAN be made Rabet-eligible from the DB:** insert/set `{{TBL_SETTING}}` row `SET_CODE='EnableRabetInquiry', SET_VALUE='1', SET_REFERENCE=<APPLICATION_ID>` (a **cache refresh** may be needed per the settings convention). This is the DB-settable mechanism the flag stories OBS-8612/ELMX-7741 describe.
- **TC-008 evidenced via 992 (FAIL):** eligible product with no packages returns error `1193 "No packages found for the given product"`, **not** the expected empty `packages:[]` (OBS-UC086B postcondition). Same empty-collection-returns-error defect family as TC-011 (`1191`).
- **TC-005 reconsidered:** eligibility IS the `EnableRabetInquiry` flag; a product with it absent/0 -> `1005` (E1 "not-exist" behaviour observed), modulo the wording gap ("The Application Not Exists" vs AC "ProductCode does not exist").
- **Use `ProductCode 999`** for happy paths. Product codes seen live: 116, 227, 999, 002 (all exist). `{{TBL_APPLICATIONS}}.APPLICATION_ID` is an **identity** column (cannot set the product code directly on insert).

### DB schema for empty-collection test data (BPQA)
- Product master `{{TBL_APPLICATIONS}}` (`APPLICATION_ID` = Product Code, `APPLICATION_ACTIVITY`).
- Product->package `{{TBL_APPLICATIONS_PACKAGES}}` (`APPLICATION_ID`,`PACKAGE_ID`,`PACKAGE_ACTIVITY`); package master `{{TBL_PACKAGES}}` (`PACKAGE_ID`,`PACKAGE_EXTERNAL_ID`,`PACKAGE_ACTIVITY`).
- Package->service `{{TBL_PACKAGES_SERVICES}}` (`PACKAGE_ID`, **`SERVUCE_ID`** [sic], `PACKAGE_ACTIVITY`).
- Service->price list `{{TBL_SERVICE_PRICE_LIST}}` (`APPLICATION_SERVICE_ID`) -> `{{TBL_PRICE_LIST}}` / `{{TBL_PRICE_LIST_DETAILS}}`.
- **Found (empty-collection rows exist in DB):** products with no active package = `199, 999000046, 999000061, 999000074, 999000080`; package with no active service = product `901` / `Package_ID testpackage01` (also 910, 5, 556, 5559).

### Why TC-008/011/012 can't be sourced from the DB (definitive, 2026-09-14)
Rabet **eligibility is NOT a BPQA column** — searched `%RABET%/%INQUIR%/%MARKET%/%ELIGIB%/%ALLOW%/%WHITELIST%/%CONSUMER%/%INTEGRAT%` (0 hits); comparing eligible `999` vs ineligible `199` shows no eligibility flag (only description/activity/created-by differ). Eligibility is the **3scale gateway subscription** of the Rabet app (app-Id {{rabetAppId}}) — external to the billing DB. Confirmed: **only product `999` is eligible** (its base-serials `999000046` etc. are NOT — all give "The Application Not Exists"), and `999` is fully populated (has packages, services, price lists). So the *empty-collection* rows exist in the DB, but none belongs to a Rabet-eligible product -> **TC-008/011/012 are not sourceable from the DB alone.**
**To unblock:** either configure a Rabet-eligible product with an empty child (e.g. add an empty package to `999`, or subscribe an empty product like `199` to the Rabet app in 3scale), or obtain the Rabet app's full eligible-product list and intersect it with the empty-collection rows above.

## Test results (2026-09-14, Rabet app-Id {{rabetAppId}}) — Plan OBS-11680 / cases OBS-11681..11692
| TC | Scenario | Result |
|---|---|---|
| 001 | Products, 999 | PASS — `applicationCode/NameEn/Ar` + **`applicationDescEn/Ar`** (descriptions ✓) |
| 002 | missing Product_Code | validation fires; "ProductCode is missing in request Header" (5001) |
| 003 | length 2 ("99") | validation fires; "ProductCode is invalid format" (5002) |
| 004 | non-existent (234) | "The Application Not Exists" (1005) |
| 005 | ineligible existing (199) | "The Application Not Exists" (1005) — E1 behavior confirmed |
| 006 | Rabet creds + eligibility | PASS — creds authenticate; 999 in / 199 out (eligibility applied) |
| 007 | Packages, 999 | PASS — packageCode/EnglishName/ArabicName/CreationDate |
| 009/010 | Services + all services (TC-009/010) | **PASS on `/service/list`** — see corrected mechanics below |

> ⚠️ **CORRECTION (2026-09-14, cont. #2).** My first cont. pass tested the WRONG endpoint (`/service/details`, from the collection/KB) and drew wrong conclusions (old F4/F5/F6 retracted). The **spec (OBS-11391)** endpoint is **`GET /api/service/list/{packageId?}`** (optional **route** param = package external code) — re-tested below against OBS-11391 + the 12 ACs (OBS-11681…11692, pulled via `jira.py`).

## CORRECTED endpoint mechanics — services & price list (verified 2026-09-14 #2, Rabet {{rabetAppId}}, ProductCode 999)
- **Services list:** `GET /billing/v3/api/service/list` → **all** services for the product (~1.03 MB on 999). Each service: `serviceCode, serviceEnglishName, serviceArabicName, serviceType(1-6), serviceTypeDesc, vatPercentage, serviceCreationDate, priceListID, priceListType(1 Range/2 Fixed), priceListItems[{price, sliceStartDate, sliceStartVolume, amountMultiplied, priceItemCreationDate}]`. **`priceListItems[]` IS present** (retracts old F5/finding #2).
- **Services by package:** `GET /billing/v3/api/service/list/{packageExternalCode}` — the **route param DOES filter** (retracts old F4/C1). Control `TEST_PKG-PD-UMRH-009` → filtered subset (1.08 KB).
- **Empty package (TC-011):** `/service/list/QA-RABET-EMPTY-011` (empty pkg 989) → `status:1, data:null`, error **`1191 "No services found for the given product."`** — **NOT** the expected empty `services:[]`. Also: spec (OBS-11391) lists `1180 NoServicesFound`, not 1191; and message says "product" though a **package** was filtered.
- **No-price service (TC-012):** a service with 0 active price lists (`NEWFARESSSS`) is **OMITTED** from `/service/list` (not returned with `priceListItems:[]`). Consistent with spec BR "one entry per (service × price list)" → **conflicts with TC-012/E4** which expects the service present with empty items.
- **Note (design vs built):** OBS-11391 flags that the approved design used a query param `?packageId=`; the build uses a **route param**. Also the **test cases TC-009…012 name `/service/details`+Package_ID**, but `/service/details` is a *different* endpoint that does NOT package-filter (path there = a *service* code → `1190`). Three contracts in the docs; as-built is `/service/list/{packageId}`.

## CORRECTED per-AC results (tested against OBS-11391 + OBS-11681…11692)
| TC | Expected (AC) | Observed | Verdict |
|---|---|---|---|
| 001 | product fields `productCode/productEnglishName/productArabicName/productAR_Description/productENG_Description` | `applicationCode/applicationNameEn/applicationNameAr/applicationDescEn/applicationDescAr` | **FAIL** (field names; API matches OBS-11391, not UC086A) |
| 002 | missing PC → "ProductCode is required" | `5001 "ProductCode is missing in request Header"` | **FAIL** (wording) |
| 003 | len <3/>9 → "ProductCode does not exist" | len2 → `5002 "ProductCode is invalid format"` (len10 not separately run) | **FAIL** (wording/behavior) |
| 004 | non-existent → "ProductCode does not exist" | `1005 "The Application Not Exists"` | **FAIL** (wording) |
| 005 [Prov] | ineligible (flag off) → "ProductCode does not exist" | `EnableRabetInquiry` flag not in BPQA; 199→`1005` | **NOT-EVIDENCED/Provisional** (flag not built; ELMX-7741/OBS-8612 open) |
| 006 (Sec) | non-Rabet creds → rejected | not tested (no non-Rabet cred); prior: non-Rabet billing creds returned data | **NOT-EVIDENCED** (blocked; only-Rabet likely not enforced) |
| 007 | packages `packageCode/packageEnglishName/packageArabicName/packageCreationDate` | matches | **PASS** |
| 008 | product w/ no packages → `packages:[]` | not testable (no eligible empty product) | **NOT-EVIDENCED** (blocked) |
| 009 | services w/ price list (Package_ID set) | present on `/service/list` incl. `priceListItems[]` | **PASS on `/service/list`** (field `amountMultiplied` vs AC `isQuantitative`; endpoint name differs) |
| 010 | Package_ID null → all services | `/service/list` returns all | **PASS on `/service/list`** |
| 011 | package w/ no services → `services:[]` | error `1191`, data null | **FAIL** (error not empty array; code≠1180; msg says product) |
| 012 | service w/ no price list → returned w/ `priceListItems:[]` | service OMITTED from list | **FAIL / doc-conflict** (matches spec BR, contradicts TC-012/E4) |

**Test data kept:** empty package `QA-RABET-EMPTY-011` (PACKAGE_ID 989, APPLICATION_PACKAGE_ID 992) on 999 — **useful after all** (evidences TC-011 on `/service/list`). Kept per QA decision; rollback SQL on file.

## Findings vs OBS-UC086 spec (CORRECTED)
1. **Products field names differ from UC086A / TC-001:** API `applicationCode/...`, TC/UC086A `productCode/...`. Also OBS-11391 (tech task) itself uses `applicationCode/...`, so the two docs conflict; API follows OBS-11391.
2. ~~Services shape differs / no priceListItems~~ **RETRACTED** — that was the wrong endpoint. `/service/list` returns `priceListItems[]` per OBS-11391.
3. **Error-message wording differs (TC-002/003/004):** "ProductCode is missing in request Header" (want "ProductCode is required"); "ProductCode is invalid format"; "The Application Not Exists" (want "ProductCode does not exist"). Same pattern as OBS-9911.
4. **Endpoint-name conflict:** TC-009…012 say `/service/details`+Package_ID; OBS-11391 says `/service/list/{packageId}`; design said `?packageId=`. As-built = route param on `/service/list`.
5. **Empty-collection semantics:** TC-008/011/012 expect empty arrays `[]`; API returns error `1191` (TC-011) or omits the row (TC-012). Needs PO decision.
6. **Error-code drift:** empty services → `1191`, spec says `1180 NoServicesFound`.
