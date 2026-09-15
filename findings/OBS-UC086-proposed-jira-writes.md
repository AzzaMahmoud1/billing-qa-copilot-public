# PROPOSED Jira writes — OBS-UC086 execution (review before I send anything)

Nothing below is sent yet. All reads done via jira.py REST (MCP is down). Rabet app-Id {{rabetAppId}}, ProductCode 999, tested 2026-09-14.

## A. Evidence comment to add to each of the 12 test cases (12 comments)

Format per comment (Jira wiki):
`*QA Execution — 2026-09-14 (Rabet app-Id {{rabetAppId}}, ProductCode 999)* / *Result* / *Endpoint* / *Observed* / *Expected (AC)* / *Evidence* / *Defect*`

- **OBS-11681 (TC-001)** — *Result:* FAIL. *Endpoint:* GET /billing/v3/api/product/list. *Observed:* HTTP 200; fields `applicationCode, applicationNameEn, applicationNameAr, applicationDescEn, applicationDescAr`. *Expected:* `productCode, productEnglishName, productArabicName, productAR_Description, productENG_Description`. *Note:* OBS-11391 itself uses `application*` → doc conflict. *Defect:* OBS-BUG-4 (clarification).
- **OBS-11682 (TC-002)** — *Result:* FAIL. *Endpoint:* GET /product/list (no ProductCode). *Observed:* `5001 "ProductCode is missing in request Header"`. *Expected:* "ProductCode is required". *Defect:* OBS-BUG-2.
- **OBS-11683 (TC-003)** — *Result:* FAIL. *Endpoint:* GET /product/list, ProductCode="99" (len 2). *Observed:* `5002 "ProductCode is invalid format"`. *Expected:* "ProductCode does not exist". (len 10 not separately run.) *Defect:* OBS-BUG-2.
- **OBS-11684 (TC-004)** — *Result:* FAIL. *Endpoint:* GET /product/list, ProductCode=999888888 (well-formed, non-existent). *Observed:* `1005 "The Application Not Exists"`. *Expected:* "ProductCode does not exist". *Defect:* OBS-BUG-2.
- **OBS-11685 (TC-005)** — *Result:* NOT-EVIDENCED (Provisional). `EnableRabetInquiry` flag not present in BPQA; ineligible 199 → `1005`. Flag stories ELMX-7741 / OBS-8612 Open. E1 mechanism not verifiable.
- **OBS-11686 (TC-006)** — *Result:* NOT-EVIDENCED (blocked). Needs a valid non-Rabet app credential (unavailable). Related: ClientKey validation works (wrong ClientKey → `5008 "Invalid ClientKey"`), but the key is the shared Billing_Client_Key, and an earlier check showed non-Rabet billing creds also returned catalog data → only-Rabet possibly not enforced.
- **OBS-11687 (TC-007)** — *Result:* PASS. *Endpoint:* GET /package/list. *Observed:* HTTP 200; `packageCode, packageEnglishName, packageArabicName, packageCreationDate` (matches AC).
- **OBS-11688 (TC-008)** — *Result:* NOT-EVIDENCED (blocked). Needs an eligible product with zero packages; only 999 is eligible and it has packages.
- **OBS-11689 (TC-009)** — *Result:* PASS (on `/service/list`). *Observed:* services with `serviceCode, serviceEnglishName, serviceArabicName, serviceType(1-6), serviceTypeDesc, vatPercentage, serviceCreationDate, priceListID, priceListType, priceListItems[{price, sliceStartDate, sliceStartVolume, amountMultiplied, priceItemCreationDate}]`. *Note:* AC names `/service/details`+Package_ID; the working endpoint is `/service/list/{packageId}` (route param) per OBS-11391. Field `amountMultiplied` vs AC `isQuantitative`.
- **OBS-11690 (TC-010)** — *Result:* PASS (on `/service/list`). No packageId → all product services returned (~1.03 MB).
- **OBS-11691 (TC-011)** — *Result:* FAIL. *Endpoint:* GET /billing/v3/api/service/list/QA-RABET-EMPTY-011 (empty package on 999). *Observed:* HTTP 200 envelope `status:1, errorsCount:1, data:null`, error `{key:1191, message:"No services found for the given product."}`. *Expected:* HTTP 200 `"services": []`. *Defect:* OBS-BUG-1.
- **OBS-11692 (TC-012)** — *Result:* FAIL / doc-conflict. *Observed:* a service with no active price list (e.g. `NEWFARESSSS`) is OMITTED from `/service/list` (absent from full response). *Expected:* service returned with `"priceListItems": []`. *Note:* matches spec BR "one entry per (service × price list)" → contradicts TC-012/E4. *Defect:* OBS-BUG-3 (clarification).

## B. Bugs to create (type 10004), each `Blocks` OBS-11680 + linked to its TC

### BUG-1 (functional) — blocks OBS-11680, relates OBS-11691
*Summary:* `[Product Catalog] Retrieve Services — empty package returns error 1191 instead of empty services array`
*Priority:* Medium
*Description:*
```
*Steps*
1. As Rabet (app-id {{rabetAppId}}), GET /billing/v3/api/service/list/{packageExternalCode} for an eligible product with a package that has zero active services.
   Example: ProductCode 999, package QA-RABET-EMPTY-011.

*Actual Result*
 * HTTP 200; body.status=1, errorsCount=1, data=null
 * error: { key: 1191, message: "No services found for the given product." }

*Expected Result*
 * HTTP 200 with "services": [] (empty array) — per OBS-8527-TC-011 / OBS-UC086C Exception E3.
 * Secondary: OBS-11391 specifies error 1180 (NoServicesFound), not 1191; and the message says "product" although the filter was a package.

*CURL*
{code:java}
curl --location 'https://<Billing_3scale>/billing/v3/api/service/list/QA-RABET-EMPTY-011' \
--header 'Content-Type: application/json' \
--header 'ProductCode: 999' \
--header 'ClientKey: <Billing_Client_Key>' \
--header 'app-id: {{rabetAppId}}' \
--header 'app-key: <Rabet app-key — from env, not pasted>'
{code}
```

### BUG-2 (functional) — blocks OBS-11680, relates OBS-11682/11683/11684
*Summary:* `[Product Catalog] product/list validation & not-exist messages do not match specified strings`
*Priority:* Low
*Description:*
```
*Steps*
1. GET /billing/v3/api/product/list with NO ProductCode header.
2. GET /billing/v3/api/product/list with ProductCode length 2 ("99").
3. GET /billing/v3/api/product/list with a well-formed non-existent code (e.g. 999888888).

*Actual Result*
 * 1 -> 5001 "ProductCode is missing in request Header"
 * 2 -> 5002 "ProductCode is invalid format"
 * 3 -> 1005 "The Application Not Exists"

*Expected Result* (OBS-UC086A / TC-002/003/004)
 * 1 -> "ProductCode is required"
 * 2 & 3 -> "ProductCode does not exist"
```

### BUG-3 (Business Clarification) — blocks OBS-11680, relates OBS-11692  [FILE ONLY IF YOU WANT]
*Summary:* `[Product Catalog] Business Clarification — service with no price list omitted vs expected empty priceListItems`
*Priority:* Low
*Description:* TC-012/E4 expects a no-price service returned with "priceListItems": []; the API omits the service entirely, consistent with spec BR "one entry per (service × price list)". Which is canonical?

### BUG-4 (Business Clarification) — blocks OBS-11680, relates OBS-11681  [FILE ONLY IF YOU WANT]
*Summary:* `[Product Catalog] Business Clarification — product field names applicationCode vs productCode`
*Priority:* Low
*Description:* TC-001/OBS-UC086A expects productCode/productEnglishName/productArabicName/productAR_Description/productENG_Description; API (and OBS-11391) return applicationCode/applicationNameEn/applicationNameAr/applicationDescEn/applicationDescAr. Reconcile the docs or fix the API.

## C. Plan
- Add each created bug to plan OBS-11680 via a **Blocks** link (bug → plan).
- Post the 12 evidence comments (A) on the 12 cases.

## Open decisions
1. **Assignee** for the bugs? (default azzaibrahim, or the dev/PO)
2. **Which bugs** to file? (BUG-1 + BUG-2 only, or also the two Business-Clarification items BUG-3/BUG-4)
3. **app-key in CURL:** I used a placeholder (existing OBS bugs paste the real Rabet key). Keep placeholder, or you add the real key?
