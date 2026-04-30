# sst/opencode #25036 — test: port instance HttpApi path/vcs read coverage to Effect

- **URL:** https://github.com/sst/opencode/pull/25036
- **Head SHA:** `7a7979ea9b166e27cdeee16e608def39849b9004`
- **Files:** `packages/opencode/test/server/httpapi-instance.legacy.test.ts` (new, +138), `packages/opencode/test/server/httpapi-instance.test.ts` (rewritten, +73/-154)
- **Verdict:** `merge-after-nits`

## What changed

Splits the single legacy bun-test instance HttpApi suite in two:

1. **Preserved the legacy contract.** A new `httpapi-instance.legacy.test.ts` keeps the bun-test-driven Hono-bridge coverage verbatim (catalog reads, `POST /project/git/init`, `PATCH /project/{id}`, `POST` instance dispose). This is the safety net — the rewrite below does not have to perfectly preserve every existing assertion because the legacy file still asserts them.
2. **New Effect-native suite.** `httpapi-instance.test.ts` uses `it.live` + `Effect.gen` + `HttpClient` + `HttpClientRequest.pipe(directoryHeader(dir), HttpClient.execute)` and exercises path / vcs / vcs-diff read endpoints with `concurrency: "unbounded"` parallelism, then asserts the diff body contains the file just written via `FileSystem.writeFileString`.

## Why it's right

- **Two-file split is the right migration shape.** The "rewrite the test suite alongside the API surface refactor" pattern almost always loses coverage in the swap because the new shape doesn't *quite* express the legacy assertions. Keeping the bun-test legacy suite as `*.legacy.test.ts` until every endpoint has a parallel Effect-native test is the conservative migration — bisect blame stays meaningful, and a regression on a not-yet-ported endpoint surfaces in legacy.
- **`it.live` choice is correct.** The new tests touch the real filesystem (`fs.writeFileString(path.join(dir, "changed.txt"), "hello")`) and a real git worktree (`tmpdirScoped({ git: true })`). `it.live` runs against the real Effect runtime rather than a TestClock-substituted one, so async-fs and child-process-git timing are realistic.
- **`concurrency: "unbounded"` on three independent reads is fine.** Path / VCS / VCS-diff are all read-only against the same scoped tmpdir, with no shared mutable state in the handler chain. Running them in parallel both speeds the test and asserts the workspace-routing middleware doesn't carry hidden per-request mutable state.
- **The diff assertion locks the right invariant.** `expect.objectContaining({ file: "changed.txt", additions: 1, status: "added" })` against the parsed VCS-diff JSON checks both that workspace-routing resolved to the right repo *and* that the diff renderer reflects the in-test write — a stronger end-to-end assertion than just status-200.

## Nits

1. **Legacy file naming sets a precedent without a follow-through plan.** `httpapi-instance.legacy.test.ts` is fine for one PR, but with no tracking issue or `// TODO(#NNN): port to Effect, then delete this file` comment at the top, `*.legacy.test.ts` will accumulate as more endpoints get ported piecemeal and become indistinguishable from "tests we forgot about". A one-line comment at `legacy.test.ts:1` linking to a "port the rest" tracking issue would make the split intentional rather than vestigial.

2. **Dispose is in legacy only, not in the new suite.** The legacy `serves instance dispose through Hono bridge` test (lines 119-138 of legacy) asserts the `server.instance.disposed` event fires through `GlobalBus`. The new Effect suite covers reads only — there's no Effect-native equivalent for the dispose lifecycle event yet. That's fine for this PR's scope, but the `it.live` shape needs a reusable `waitForBusEvent(predicate)` helper before dispose can be ported. Worth flagging in the PR description so the next porter doesn't reinvent it.

3. **`tmpdirScoped({ git: true })` is the new fixture verb.** It's not introduced in this diff, presumably lives in `test/lib/effect.ts` — but the legacy file uses `tmpdir({ config: { formatter: false, lsp: false } })` from a different module. The two fixtures differ on whether `git: true` runs `git init` and on the config-disabling defaults. A second PR consolidating these into a single fixture surface (probably `tmpdirScoped` everywhere) would prevent the next porter from picking the wrong one and silently changing the test's environment.

4. **No assertion that `vcsDiff` returns 200 *before* the file write.** The current shape writes `changed.txt`, then concurrently reads paths / vcs / vcs-diff. If the handler accidentally blocks until *some* diff exists, the parallel ordering may hide it. A separate `it.live` that exercises `vcsDiff` against an empty worktree (asserting `200` + empty array) would lock that branch.

## Risk

Low. Test-only change with the legacy file preserving the previous assertion set verbatim. The only way this regresses production behavior is if a future PR deletes `httpapi-instance.legacy.test.ts` before all endpoints are ported — and the file naming makes that mistake easy. Calling out the "this is intentionally redundant for the migration window" intent in a comment would lower that risk meaningfully.
