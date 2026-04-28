# sst/opencode #24717 — fix(httpapi): document tui bad request responses

- **Head SHA**: `964101d07c2b13260bae995c9b76e9b53fef6dab`
- **State**: OPEN (stacked on top of #24716)
- **Author**: kitlangton
- **Size**: +16 / -2 across 2 files

## Files changed

- `packages/opencode/src/server/routes/instance/httpapi/tui.ts` (+4/-1)
- `packages/opencode/test/server/httpapi-tui.test.ts` (+12/-1)

## What it does

Brings the experimental Effect HttpApi TUI endpoints' OpenAPI surface to parity with the legacy Hono router by declaring `HttpApiError.BadRequest` as a possible error variant on the four POST endpoints whose JSON payload validation can fail (`appendPrompt`, `executeCommand`, `publish`, `selectSession`). Without these declarations the regenerated SDK and the published OpenAPI doc don't list the 400 response, leaving SDK consumers without typed error handling for the most common failure mode (malformed JSON body).

## Specific observations

- **Three endpoints get a single-error declaration, one gets a list.** `appendPrompt` (`:64`), `executeCommand` (`:117`), and `publish` (`:138`) are pure additions of `error: HttpApiError.BadRequest`. `selectSession` (`:149`) is the more interesting case — it already declared `error: HttpApiError.NotFound` (the session-doesn't-exist 404) and the change rewrites that to `error: [HttpApiError.BadRequest, HttpApiError.NotFound]`. Effect HttpApi's union-error syntax is array-of-variants, so this is the right shape, and *order matters* only for codegen output ordering, not behavior. The legacy Hono router presumably documents both 400 and 404 on this endpoint, so maintaining that union is what closes the parity gap rather than replacing one with the other.
- **Test design is OpenAPI-introspective, not request-driven.** At `test/server/httpapi-tui.test.ts:43-50` the test calls `Server.openapi()` (legacy router's published doc) and `OpenApi.fromApi(TuiApi)` (Effect's reflected doc), then walks all four `TuiPaths` and asserts both docs contain a 400 response on the POST. This is the right test for a "documentation parity" PR — it verifies the *contract surface* not the runtime behavior, which is exactly what changed. A request-driven test would have been a stronger end-to-end pin (send a malformed body, assert a 400 with a structured error envelope) but would also have duplicated coverage that probably already exists for the legacy router.
- **Test coverage gap on the `selectSession` 404 variant.** The new test pins that *400* is documented on all four endpoints, but the `selectSession` change also asserts (implicitly) that *404* still appears alongside it. Without an explicit `expect(legacy.paths[selectSession].post?.responses?.[404]).toBeDefined()` and the Effect-side equivalent, a future "simplification" that drops `HttpApiError.NotFound` from the union would silently regress the 404 declaration with no test signal. The test as written checks 400-presence but doesn't pin that the union-error rewrite preserved the existing 404.
- **`paths[path].post?.responses?.[400]` chain at `:48-49`** uses optional chaining all the way down — if a path is missing entirely from one of the docs, `responses` is `undefined` and `[400]` is `undefined` and `toBeDefined()` fails with a generic message. A custom assertion message naming the path (`expect(...).toBeDefined({ message: \`expected 400 on ${path}\`})`) would make a regression on, say, `appendPrompt` but not the others, much easier to diagnose. Probably not worth blocking on.
- **Stacked PR dependency**: PR description notes this is stacked on #24716 (the sync seq validation parity fix). The two PRs share a pattern — declare `HttpApiError.BadRequest` so payload validation surfaces as a documented 400 — but operate on disjoint endpoint sets, so review order doesn't gate correctness. If #24716 lands first and the rebase is clean, this PR's `tui.ts` changes shouldn't conflict.
- **`HttpApiError.BadRequest` import is already present** at the top of `tui.ts` (it's used by the existing `selectSession` `NotFound` declaration via the same `effect/unstable/httpapi` import), so the diff doesn't need an import change. Worth confirming the regenerated SDK at `packages/sdk/js` isn't included in this PR — typically a separate codegen step would produce the SDK delta. The PR description's "verification" mentions `bun typecheck` from `packages/sdk/js` but no SDK file changes appear in `files`, suggesting either (a) the codegen output is up-to-date already, or (b) the PR author is leaving SDK regeneration to a follow-up. Either is fine but worth a maintainer call before merge.

## Verdict

**merge-after-nits** — surgical OpenAPI parity fix that correctly declares `HttpApiError.BadRequest` on the four payload-validating TUI endpoints and uses the right Effect HttpApi union-error syntax for the `selectSession` case where 400 must coexist with the existing 404. The introspective test pins the parity invariant the PR is shipping. Two nits before merge: an explicit assertion that `selectSession`'s 404 declaration survived the union rewrite, and confirmation of whether the SDK regeneration step needs to land alongside this PR or in a follow-up.
