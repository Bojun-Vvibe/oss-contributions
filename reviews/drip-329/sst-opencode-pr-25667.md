# sst/opencode #25667 — research: delete Hono backend (do not merge)

- SHA: `1c3ff63927876e3bc1ab5c09c46d5b24136e83ce`
- State: OPEN, +37/-8,881 across 58 files
- PR body explicitly says **do not merge**. Treated as a scope-spike / blast-radius visualization rather than a real merge candidate.

## Summary

Research draft removing the Hono backend wholesale to expose the size of the cliff for the eventual "HttpApi is the only backend" PR. Deletes 14 Hono per-feature route files, the Hono shell (middleware/error/fence/proxy/workspace), the adapter layer (`adapter.ts`, `adapter.bun.ts`, `adapter.node.ts`), the backend selection layer (including `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI` and `HTTPAPI_DEFAULT_ON_CHANNELS`), all `httpapi-{bridge,parity,json-parity}` and `httpapi-instance.legacy` parity tests, the `script/httpapi-exercise.ts` cross-backend diff harness, and the Hono dependencies from `package.json` (`hono`, `hono-openapi`, `@hono/node-server`, `@hono/node-ws`, `@hono/standard-validator`, `@hono/zod-validator`).

## Notes

- `packages/opencode/src/server/server.ts:+20/-203` — `Server` collapses to HttpApi-only `listen()` + `openapi()`. The deletion is the load-bearing change; everything else cascades. Worth confirming on the eventual real PR that no external SDK consumer imports `Server.openapiHono()` (PR body flags this as a known-cliff item under "OpenAPI generation").
- `packages/opencode/src/cli/cmd/generate.ts:+4/-19` — drops the `--hono` / `--httpapi` flags; always uses HttpApi spec. Cleaner CLI surface, but if the SDK-generation step in CI passes either flag explicitly that pipeline breaks.
- `packages/opencode/src/server/routes/instance/{config,event,experimental,file,mcp,permission,project,provider,pty,question,session,sync,trace,tui}.ts` — 14 route files deleted (e.g., `session.ts` -1,124, `experimental.ts` -419, `pty.ts` -340, `tui.ts` -387). Confirms the PR body's "HttpApi is genuinely a feature-superset" claim only if `bun turbo typecheck` passing across all 13 packages also catches missing route handlers — it does (each deleted route used to be reachable through `routes/instance/index.ts`'s mount, also deleted, so dead-code at the call site).
- `packages/opencode/src/server/backend.ts` (-32) and `packages/core/src/flag/flag.ts` (-13) — removes the runtime backend selector and `OPENCODE_EXPERIMENTAL_HTTPAPI` flag wholesale. PR body's note that the real merge needs a one-liner `flag.ts` change to flip default to HttpApi for stable channels is correct; this draft skips the in-between staged-rollout step.
- `packages/opencode/script/httpapi-exercise.ts` (-2,014) — deletion of the cross-backend diff harness. This is the primary tool for proving HttpApi parity; deleting it before the real merge means losing the regression check. Recommend the real merge keep this harness around for at least one release after Hono is gone, even if it just no-ops, so any HttpApi regression is caught against a known-good golden.
- `packages/opencode/test/server/httpapi-listen.test.ts:+1/-21` — drops the `keeps PTY websocket tickets optional when server auth is disabled` test. PR body's "Known cliff item #1" explicitly calls this out: the original Hono path skipped ticket validation when `OPENCODE_SERVER_PASSWORD` was unset; HttpApi's `ptyConnectRoute` lacks the equivalent bypass. Two valid resolutions stated. The right call is to add the bypass to HttpApi rather than drop the test — dropping silently changes auth semantics for users running without server password.
- 15 `httpapi-*` test files (`httpapi-config.test.ts`, `httpapi-cors.test.ts`, etc.) drop their Hono parametric branches. Mechanical and correct.
- `packages/opencode/test/server/{instance-bootstrap-regression,httpapi-tui,trace-attributes}.test.ts` — three test files deleted entirely because they were Hono-specific. PR body acknowledges this. The real merge needs to confirm those weren't pinning HttpApi-side regressions too.
- The PR body itself is high-quality: lays out blast radius (~8.5k LOC), explicitly enumerates 4 cliff items (no-auth PTY parity, CI checks, OpenAPI generation, untouched dependents `tui/worker.ts` and `acp.ts`), and ends with a "do not merge" + "rebase or reopen when stable flips" disposition. This is the right way to file an exploratory deletion.
- One thing missing from the cliff list: the SDK consumers. The OpenAPI spec generated from HttpApi may differ from the Hono-generated one in subtle ways (route ordering, parameter naming, response examples). Real merge should diff the two specs and call out non-cosmetic deltas before flipping the default.

## Verdict

`needs-discussion` — the PR is honest about its non-mergeability. The discussion artifact is valuable and the cliff-item list is the right deliverable. Concrete asks before reopening as a real merge: (a) add the no-auth PTY bypass to HttpApi rather than deleting the test, (b) keep `script/httpapi-exercise.ts` operational for one release post-flip as a regression net, (c) add a generated-OpenAPI-spec diff to the cliff item list, (d) sequence the real merge as flag-flip → bake → delete, not delete-in-one-shot.
