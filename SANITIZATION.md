# Sanitization — private → public

This repo is maintained as a **private** tree with real internal schema inline. The **public** copy is a
generated, sanitized build. Two repos, one source.

> **Note:** `tools/sanitize.py` and `knowledge-base/_private/` live **only in the private repo** — the
> public build excludes both. If you are reading this in the public repo, the commands below were run
> in the private repo to produce what you see here.

## Produce the public build
```bash
python tools/sanitize.py ../billing-qa-copilot-public
```
This copies the tree, applies the transforms below, removes private material, and runs the **release
gate**. It exits non-zero (and you must not publish) if any forbidden content remains.

Re-check an existing build without rebuilding:
```bash
python tools/sanitize.py --check ../billing-qa-copilot-public
```

## Transforms (what changes)
| Private | Public |
|---|---|
| `{{TBL_}}<X>` (any table) | `{{TBL_<X>}}` (documented placeholder) |
| real sample invoice number | `<SAMPLE_INVOICE_NO>` |
| `*{{internalHost}}` hosts | `{{internalHost}}` |
| `knowledge-base/_private/` | **removed** |
| `api-collection/environment.json` | **removed** (gitignored anyway) |
| `api-collection/collection.json` (real export) | **removed** — public ships `collection.example.json` (synthetic) instead |

Examples already use synthetic identifiers, so they need no change — only verification.

## Release gate (the hard stop)
The public build must contain **zero** hits for: `{{TBL_}}`, the real invoice number, `{{internalHost}}`, and
must not contain `knowledge-base/_private/`. The gate greps for all of these; any hit → FAIL → do not
push. A human reviews the diff and confirms data-owner sign-off before the first public push.

## What the public repo still gives a reader
The reasoning, structure, method, business rules, and runnable SQL **templates over placeholders** — so
a reader sees exactly *where* and *how* to test; they supply their own real table names via their own
(private) map. "Don't over-redact": placeholders stay meaningful (`{{TBL_INVOICE_ZATCA}}`, not
`{{TABLE_7}}`).
