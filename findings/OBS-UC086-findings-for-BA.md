# Rabet – Get Product Catalog Details (OBS-UC086): my findings vs the use case & restrictions

I went through the Rabet "Get Product Catalog Details" APIs (OBS-UC086 A/B/C) against the use case
and the restrictions. I tested everything live on product 999 as the Rabet app. A few things don't
line up with the documents — listed below. Where I couldn't open the ACs in Jira at the time, I've
said so and flagged it for you to confirm.

**1. We can't retrieve services by package.**
The use case (OBS-UC086C) says Package_ID is optional and, when provided, should return that package's
services. In the actual API there's no such filter — `/service/details` only returns all services for
the product, and if I put the package in the path it treats it as a service code and errors with
"Service does not exist under this product." I tried it as a header, as a query parameter and as a path
segment; none of them filter by package. There's a request in the collection called GetServiceByPackageV3,
but it isn't wired up (it just points at `/package/list`). Because of this I also couldn't test the
empty-package case.

**2. An empty price list comes back as an error, not an empty list.**
When a service has no price list, `/service/{code}/pricelist` returns an error ("The Price List Not
Exists", code 1007) instead of an empty result. I tested this on service NEWFARESSSS on 999 (no prices)
and compared it to Test_PD-UMRH-009 (has a price, returns normally). Can you confirm what the AC expects
here — an empty list or an error? I couldn't open OBS-11692 at the time. Also worth noting: the message
is broken — it literally reads "The Price List Not Exists - {0}", with the {0} placeholder never filled in.

**3. The response and error format don't match the documented contract.**
- Products: fields come back as applicationCode / applicationNameEn / applicationNameAr / applicationDescEn / applicationDescAr, but OBS-UC086A specifies productCode / productEnglishName / productAR_Description / productENG_Description.
- Services: the spec expects each service to carry its price info (Type, priceListID, priceListItems[]),
  but the response is flat (serviceType, rate, pricingModel) with no priceListItems — prices are only
  available from the separate `/pricelist` call.
- Error wording differs from the spec, e.g. "ProductCode is missing in request Header" vs "ProductCode is
  required", and "The Application Not Exists" vs "ProductCode does not exist".
These are all the same underlying issue: the API is using the internal billing/"application" naming and
shape instead of the "product" catalog contract we documented. Either the API matches the spec, or we
update the spec and re-baseline the ACs.

**4. Access isn't actually limited to Rabet.**
Restriction 1 says only Rabet should be able to call the catalog, with the Rabet Client Key hardcoded and
validated. The ClientKey check does work — a wrong key is rejected with "Invalid ClientKey" — but the key
being checked is the shared Billing Client Key, not a Rabet-specific one, so it doesn't make access
Rabet-only. And when I tried non-Rabet billing credentials earlier, they also returned catalog data. So
the "only Rabet" restriction isn't really enforced. I still need a valid non-Rabet app credential to prove
this definitively — can you help me get one, or confirm where the restriction is meant to be enforced (the
3scale subscription)?

**5. The "Allow Rabet inquiry" control doesn't exist yet.**
Restriction 2 describes eligibility being driven by an "Allow Rabet inquiry" flag set at onboarding
(ELMX-7741) and managed later in the New Billing Portal (OBS-8612). I couldn't find any such flag in the
billing database — right now the only thing deciding eligibility is the 3scale subscription, and only
product 999 is eligible. Both of those stories are still open, so this looks like it simply hasn't been
built yet.

**6. I couldn't confirm the in-scope product.**
The restriction says the scope is currently "Absher B2B Manual Contracts", but the only product I can
actually inquire on is 999, which looks like a QA/test product (it's full of test packages like
"NEWWWWW"). Can you confirm which product code is Absher B2B Manual Contracts so I can test against the
real in-scope product? I can also check 999's name in the DB if that helps.

**7. Ineligible and non-existent products give the same error.**
Asking about 199 (exists but not allowed for Rabet) and 234 (doesn't exist) both return the same "The
Application Not Exists". I wasn't sure if that's intended — should they be distinguishable?

**8. I couldn't cover the empty-packages case (TC-008).**
That case needs a Rabet-eligible product with no packages, and the only eligible product (999) has
packages, so I had no way to produce it. Same root cause as the eligibility point above.

A few of these I've marked as needing your confirmation (2, 6, 7, 8) because I couldn't get into Jira to
read the exact ACs at the time — the rest are confirmed against the spec text we already have. Happy to
walk through any of them.
