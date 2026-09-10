# Billing QA-Copilot — a Test Navigator for the B2 New Payment Portal

Hand it a **user story** about the billing / payment APIs. It tells you **how to test it** and —
the part other tools skip — **where to test it**: which **API endpoint family**, which
**`{{TBL_*}}` database table**, which **screen**, plus a ready **SQL side-effect probe** and the
compliance **edge cases** (ZATCA, Sadad, refunds, VAT). Every answer is cited to a seeded knowledge
base, and it says **"not covered"** instead of guessing.

> **Why this exists.** A one-time 53-minute handover (the "HO" doc) held the billing system's tacit
> knowledge — the invoice lifecycle, four pricing models, ZATCA/Sadad integration, and the exact
> tables to query to verify each path. This repo turns that into a queryable asset so a tester handed
> a new story knows *what to exercise and where to observe it* in minutes, instead of re-watching a
> recording or guessing at a table.

## 60-second quickstart

**As a Claude Code project (recommended):** open this folder in Claude Code and say:

> "Here's a new story: _As billing, issue a partial refund against an already-paid, ZATCA-cleared
> invoice._ Give me the test map."

The agent follows `CLAUDE.md` → reads `knowledge-base/` + `method/` → returns a test-map table
(capability · API family · table · screen · SQL probe · edge cases · citation) and one detailed block
per testable unit. See a full worked run in [`examples/`](examples/).

**As plain docs:** read `method/story-analysis-method.md` and walk your story through Steps 0–5, using
`knowledge-base/` as the lookup. The `.claude/` folder is ignored by non-Claude-Code users.

## What's inside

| Path | What it is |
|---|---|
| `CLAUDE.md` | The operating contract — the loop, output shape, and hard rules. |
| `knowledge-base/` | The seeded domain knowledge (glossary, data-model map, pricing models, lifecycle, integrations, business rules BR-01..BR-34, screens) + `how-to-verify/` recipes. |
| `method/` | The reusable reasoning: story-analysis procedure, test-design checklist, output template. |
| `api-collection/` | The drop-in slot for a Postman collection (input contract). Endpoints stay as families until you supply one. |
| `examples/` | Worked story → test-map runs. |
| `.claude/agents/` | A single agent definition so the repo is a drop-in Claude Code subagent. |
| `tools/sanitize.py` | *(private repo only)* Produces the sanitized public build + runs the leak release-gate. |

## The API collection (coming later)

Endpoints are intentionally left as **families** with `[API SLOT — pending collection]` markers until a
real Postman collection is dropped into `api-collection/`. Then the agent can cite concrete request
names/paths. See [`api-collection/README.md`](api-collection/README.md) for the ingest contract.

## Fidelity discipline

Everything in `knowledge-base/` is labelled **FACT** (stated in the handover), **INFERRED** (deduced,
flagged), or **GAP** (only the collection/PO can fill it). The agent never fabricates a table or
endpoint and never reports a PASS it cannot evidence. Known unknowns live in
[`knowledge-base/coverage-gaps.md`](knowledge-base/coverage-gaps.md).

## Public / private

The published (public) copy is a **sanitized** build — internal table names become documented
placeholders and `knowledge-base/_private/` is removed. See [`SANITIZATION.md`](SANITIZATION.md).
