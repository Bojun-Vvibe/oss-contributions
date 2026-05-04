# Review: sst/opencode #25166

- **Title:** docs: add missing docs for global config endpoints
- **Head SHA:** `964c32b731d4b29519dad4ad566d8a9a77c22b60`
- **Scope:** +8 / -6, single file `packages/web/src/content/docs/server.mdx`
- **Drip:** drip-339

## What changed

Pure docs addition: documents the previously-undocumented global config
HTTP endpoints exposed by the local server.

## Specific observations

- `packages/web/src/content/docs/server.mdx` — net +2 lines suggests the
  diff replaces an existing endpoint table row with a slightly expanded
  one rather than appending a new section. Confirm the rewritten row keeps
  the same anchor slug so existing deep-links from issues don't 404.
- The new endpoint description should state the exact HTTP verb and the
  request/response shape (or link to the `openapi.json`). MDX-only prose
  without a curl example is hard to verify against runtime behaviour.
- If the doc references a `/config/global` (or similar) path, double-check
  that path against `packages/opencode/src/server/*.ts` route definitions
  — historically these docs have drifted from the actual routes.
- No code or tests touched; CI cost is negligible.
- File touches only the EN doc; the ES translation under
  `docs/es/server.mdx` will silently drift. Either include a parallel ES
  edit or open a follow-up issue tagged `i18n`.

## Risks

- Docs-only, so the only failure mode is incorrectness or i18n drift.

## Verdict

**merge-after-nits** — verify the documented path matches the live route
and either mirror the change in ES or file an i18n follow-up.
