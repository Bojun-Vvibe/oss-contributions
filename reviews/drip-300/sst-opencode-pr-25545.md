# sst/opencode PR #25545 — feat(server): Server.openapi() backed by HttpApi spec, parity-checked against Hono output

- Author: kitlangton
- Head SHA: `f2f4561d1a6643686231af989f9bba38a6713e0b`
- Diff: +48 / -15 across 6 files
- Files: `packages/opencode/script/httpapi-exercise.ts`, `packages/opencode/src/cli/cmd/generate.ts`, `packages/opencode/src/server/server.ts`, `packages/opencode/test/server/httpapi-bridge.test.ts`, `packages/opencode/test/server/httpapi-tui.test.ts`, `packages/sdk/js/script/build.ts`

## Observations

1. **`server.ts:95-103` flips the default**: `Server.openapi()` now returns `OpenApi.fromApi(PublicApi)`; the legacy Hono-derived spec is preserved as `openapiHono()` for parity diffing only. The cutover is well-fenced — every consumer that wants the old behavior calls the renamed function explicitly, so this is a safe rename + default swap rather than a behavior change in disguise.
2. **`packages/sdk/js/script/build.ts:159-167` inverts the env switch**: `OPENCODE_SDK_OPENAPI=httpapi` (the new default) now runs `bun dev generate` plain; the `=hono` branch passes `--hono`. Worth confirming any CI that pinned `OPENCODE_SDK_OPENAPI=httpapi` still yields the same SDK shape — the comment claims `PublicApi` carries `matchLegacyOpenApi` so wire-shape is preserved, but a one-time `git diff` of the generated `openapi.json` before merge would close the loop.
3. **`generate.ts:30-52` adds `--hono` while keeping `--httpapi`**: `--httpapi` is now a no-op kept for backwards compatibility; `--hono` is the escape hatch. Description correctly flags `--hono` as "will be removed". Consider adding a deprecation log for `--httpapi` to surface to users still scripting against it.
4. **Parity tests at `httpapi-bridge.test.ts:222-249` and `httpapi-tui.test.ts:46` were redirected to `openapiHono()`** — meaning the parity suite continues to compare *both* backends rather than collapsing into a single self-test. Good. This is what makes the cutover defensible.
5. **`script/httpapi-exercise.ts:1509` also moved to `openapiHono()`** — consistent. The script's `effectRoutes` vs `honoRoutes` diff retains its meaning.
6. **One nit**: the JSDoc on `openapi()` (`server.ts:80-94`) is excellent for context, but `openapiHono()` only gets a one-liner. Consider moving the deletion-tracking note (`Delete once the Hono backend is removed`) into a `// TODO(remove-hono):` so it shows up in grep when the cleanup ticket lands.

## Verdict: `merge-after-nits`

Solid cutover with parity tests retained as the safety net. The `--httpapi` flag becoming a no-op deserves either a deprecation log or a clear note in the changelog so downstream scripts don't silently use stale assumptions. SDK regeneration diff should be eyeballed once before merge.
