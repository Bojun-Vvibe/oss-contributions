# sst/opencode #24548 — feat(httpapi): bridge tui routes

- Author: kitlangton (Kit Langton)
- Head SHA: `12fd08f780da80b1fd23418e40f3af7a8b91d9ac`
- Stacked on #24547 (pty bridge); +415 / −20 across 6 files
- Files: `packages/opencode/src/server/routes/instance/httpapi/tui.ts` (+286 new), `httpapi/server.ts` (+8), `instance/index.ts` (+14), `instance/tui.ts` (+13/−5), `specs/effect/http-api.md` (checklist flip), new test `test/server/httpapi-tui.test.ts` (+79).

## Specifics

- New `TuiApi` group at `httpapi/tui.ts:67-167` defines all 13 TUI endpoints with `OpenApi.annotations` identifiers and routes them through the `Authorization` middleware (`httpapi/tui.ts:168`). The schemas re-use `TuiEvent.PromptAppend.properties` / `ToastShow.properties` / `SessionSelect.properties` directly so the bridge stays in lockstep with the legacy event types — no parallel schema drift.
- `commandAliases` table at `httpapi/tui.ts:23-37` (e.g. `session_new` → `session.new`, `messages_page_up` → `session.page.up`) preserves the old snake_case wire contract while internal bus events use the dotted form. Worth a brief comment that this is for backwards compatibility with existing TUI clients, otherwise future readers will assume the dotted form is canonical and silently drop aliases.
- `tuiHandlers` (`httpapi/tui.ts:177+`) is constructed with `Layer.unwrap(Effect.gen ...)` so `Bus.Service` and `Session.Service` are resolved once at layer build time rather than per-request — correct pattern for a hot path.
- Server registration at `httpapi/server.ts:86-91` provides `Session.defaultLayer` and `Bus.layer` alongside `tuiHandlers` — note this is the only group besides `SessionApi` that needs `Session.defaultLayer`; if `Bus.layer` is already provided globally elsewhere the explicit `Layer.provide(Bus.layer)` here may double-register. Verify once in a follow-up.
- The legacy queue is shared via `nextTuiRequest` / `submitTuiResponse` exported from `../tui` — exactly the right move so the compatibility route in `instance/tui.ts` and the new Effect bridge cannot diverge. The +13/−5 in the legacy file is the export surface change.
- `specs/effect/http-api.md:329-345` checklist flips all 13 TUI bullets to `[x]` and items 12 + 13 in the remaining-PR plan, accurately reflecting the stacked PR set.
- `test/server/httpapi-tui.test.ts` (+79) is a real integration test asserting handler behaviour; not a stub.

## Concerns

- The `selectSession` endpoint at `httpapi/tui.ts:131-141` declares `error: HttpApiError.NotFound` but I'd want to confirm the handler actually maps a missing `SessionID` to that error and not a 500 — visible in lines beyond the diff window I read; flag for the maintainer to spot-check.
- No mention in PR description of feature-flag default — the `OPENCODE_EXPERIMENTAL_HTTPAPI` gating is referenced but not shown in this diff slice; assume the gate already exists from the stacked PRs.

## Verdict

`merge-after-nits` — solid, well-scoped bridge PR with real tests and a faithful schema reuse strategy. The two nits (alias-table comment, `Bus.layer` double-provide check) are non-blocking; selectSession 404-mapping should be eyeballed.

