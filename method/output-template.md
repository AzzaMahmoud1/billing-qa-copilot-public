# Output Template

## Output template (the agent fills one block per testable unit)


```
### TU-<NN>: <short objective>

**Test Objective:** <single observable assertion>
**Capability / Pricing model / Integration:** <from Step 1–2>
**Traces to:** <BR-xx / lifecycle state / glossary term>

**Preconditions:**
- <product/service registered, customer exists, invoice in state X, config Y>

**Steps:**
1. <action>
2. <action>

**API call(s):**
- [API SLOT — pending collection] <operation name>: endpoint + method + key request fields
  (known fields from source: <e.g. customerNin, serviceId, Volume, Amount, UnitPrice, BankTransfer, zatcaProfileID>)
- Expected response: <status / returned Sadad number / sequence number / error code>

**SQL verification (table + what to look for):**
- `{{TBL_}}<...>` — <columns / condition> → <what proves pass>
  (bind by INVOICE_NUMBER / INVOICE_ID; refunds & memos via RELEATED_INVOICE_ID)

**UI layer:**
- <screen> — <what the tester observes>  (or "n/a — no UI surface")

**Expected result:** <the pass condition, observable at one of the layers>

**Edge cases:** <boundary / negative / i18n / idempotency variants to spin off as sibling TUs>

**Verdict readiness:** PASS-able via <layer> | otherwise NOT-EVIDENCED until <collection/PO item>
```

---
