# Changelog

The knowledge base is a **living** asset. Record every domain fact added/changed (with its fidelity
label) and every sanitization/structure change.

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
