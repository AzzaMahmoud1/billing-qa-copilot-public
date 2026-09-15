# OBS-UC086 — full use-case traceability (source: Confluence 264631311 + 3 use-case docs)

Source read 2026-09-15: Confluence page "Get Product Catalog Details" (BIL/264631311) + Use_Case_Retrieve_Product/Package/Service_Details.docx. Ticket comments/attachments checked: **OBS-8527 and OBS-11391 have 0 comments, 0 attachments** (no hidden requirements there).

## A. Restrictions / preconditions
| Source requirement | Status | Evidence |
|---|---|---|
| Only Rabet may access (Client Key hardcoded + validated) | PARTIAL | ClientKey validated (wrong→5008); "only-Rabet" caller gate unproven (TC-006 blocked, needs non-Rabet cred) |
| Product must have **EnableRabetInquiry enabled** to be eligible | **CONFIRMED** | 992 (flag=1)→eligible (1193); 199 (no flag)→1005. Matches spec exactly. |
| Scope = Absher B2B Manual Contracts | UNVERIFIED | 999/992 are QA products; not confirmed = Absher |
| Package must have ≥1 active service (assumption) | NOTE | but E3 explicitly handles "package with no services" → empty list |

## B. Input validation (Basic Flow step 1)  — spec wording vs API
| Rule (spec) | Expected msg (spec) | API | Verdict |
|---|---|---|---|
| ProductCode required | "ProductCode is required" | 5001 "ProductCode is missing in request Header" | **BUG** OBS-11694 |
| Length 3–9 | "ProductCode does not exist" | 5002 "ProductCode is invalid format" | **BUG** OBS-11694 |
| Exists+Active+EnableRabetInquiry else not-exist | "ProductCode does not exist" | 1005 "The Application Not Exists" | **BUG (wording)** OBS-11694 |

## C. Output fields — spec name vs API name  (⇒ field-contract bug, broaden OBS-11696)
| Level | Spec field | API field | Match? |
|---|---|---|---|
| Product | ProductCode / ProductEnglishName / ProductArabicName | applicationCode / applicationNameEn / applicationNameAr | **NO** |
| Product | (spec lists no description) | applicationDescEn / applicationDescAr | API extra |
| Package | Code / EnglishName / ArabicName / CreationDate | packageCode / packageEnglishName / packageArabicName / packageCreationDate | **NO** (names differ) |
| Service | ServiceCode | serviceCode | yes |
| Service | EnglishName / ArabicName / Type / CreationDate | serviceEnglishName / serviceArabicName / serviceType / serviceCreationDate | **NO** (prefix differs) |
| Service | VatPercentage / PriceListID / PriceListType | vatPercentage / priceListID / priceListType | yes |
| Service | **PriceListCreationDate** | ??? | **GAP — not verified in response (live test needed)** |
| Service | (spec none) | serviceTypeDesc | API extra |
| PriceItem | Price / SliceStartDate / SliceStartVolume / PriceItemCreationDate | price / sliceStartDate / sliceStartVolume / priceItemCreationDate | yes |
| PriceItem | **isQuantitative (0=No,1=Yes)** | **amountMultiplied (0=Slice,1=PerUnit,2=Accumulative)** | **NO — name + value-set differ (BUG candidate)** |

## D. Exception flows — spec vs API  (⇒ the empty-collection bugs, now source-confirmed)
| Flow | Spec expected | API observed | Verdict |
|---|---|---|---|
| E1 product out of scope (flag disabled) | error "ProductCode does not exist" | 1005 "The Application Not Exists" | BUG (wording) OBS-11694 |
| **E2 no active packages** | **Empty Package List** | 1193 "No packages found for the given product" (error) | **BUG** — TC-008, NOT yet filed |
| **E3 package with no active services** | **Empty Services List** | 1191 "No services found for the given product" (error) | **BUG** OBS-11693 |
| **E4 service with no active price items** | **Empty Price List Items** | service omitted from list (no empty array) | **BUG** OBS-11695 (currently filed as "clarification" — should be a defect: spec is explicit) |

## E. Business & system rules — coverage
| Rule | Status |
|---|---|
| Flag disabled then enabled → Rabet can inquire | CONFIRMED (992 enabled works; toggling implied) |
| VatPercentage inheritance (SERVICE_INHERIT_VAT_FROM_PARENT → product VAT) | **GAP — not tested (live)** |
| Dynamic services follow E4 (empty price items) | **GAP — not tested (live)** |
| "Items limit on API" (result cap / pagination) | **GAP — not tested (live)** |
| Date format `2024-03-05T01:00:00` | PASS (observed `2024-10-01T00:00:00`) |
| Type enum 1–6 / PriceListType 1–2 | PARTIAL (saw type 1, priceListType 2) |
| SliceStartDate not working on Subscription (spec notes an open bug) | NOTE — re-verify |

## Gaps still needing LIVE execution (blocked: screen locked at write time)
1. **PriceListCreationDate** present in `/service/list`? (spec field C)
2. **isQuantitative vs amountMultiplied** — confirm on a service with amountMultiplied=2 (value-set mismatch).
3. **VAT inheritance** — a service with SERVICE_INHERIT_VAT_FROM_PARENT=1 returns the product VAT.
4. **Items limit on API** — is there a cap / pagination?
5. **Dynamic-pricing service** → E4 empty-price behaviour.
6. **TC-005 flag-toggle** — set EnableRabetInquiry 0→1 on a product and confirm eligibility flips (DB write, gated).

## Bugs implied vs filed
- Filed: OBS-11693 (E3), OBS-11694 (input msgs incl. E1), OBS-11695 (E4 — reclassify to defect), OBS-11696 (Product field names — **broaden to Package+Service** per section C).
- **Not yet filed:** E2/TC-008 (empty packages → 1193 not empty list); isQuantitative/amountMultiplied mismatch; PriceListCreationDate (if missing).
