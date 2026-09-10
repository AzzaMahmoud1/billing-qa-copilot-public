# Changelog

The knowledge base is a **living** asset. Record every domain fact added/changed (with its fidelity
label) and every sanitization/structure change.

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
