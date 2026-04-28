# PR #24861 — fix(bash): release parsed syntax trees

- **Repo:** sst/opencode
- **Link:** https://github.com/sst/opencode/pull/24861
- **Author:** Hona (Luke Parker)
- **State:** OPEN
- **Head SHA:** `71baf86081c6eceebb29200b8951e5ee056c9ad7`
- **Files:** `packages/opencode/src/tool/bash.ts` (+12/-5)

## Context

`web-tree-sitter` is a WASM build of tree-sitter. The JS shim wraps native (Emscripten) memory: every `parser.parse(command)` allocates a `Tree` outside the JS heap and only frees it when `tree.delete()` is called. Dropping the JS reference does **not** free the native allocation because there's no FinalizationRegistry hook in the published shim. The bash permission scanner runs on every `bash` tool call, so a session that runs hundreds of shell commands will steadily climb in RSS until the Bun process is killed by the OS.

## What changed

The author wraps the parse in `Effect.acquireRelease` inside `Effect.scoped` so `tree.delete()` is invoked on the `Tree` after `collect`/`ask` complete and before the actual command executes. The diff returns the `Tree` from the parse step, reads `tree.rootNode` while the scope is alive, and lets the runtime call the release on success, failure, or interruption. That's the right shape for this kind of resource — it generalises across the three early-exit paths (parse failed, parse succeeded, permission rejected) without sprinkling try/finally everywhere.

## Evidence

The benchmark in the PR description is the right one: 50,000 parses, `Bun.gc(true)` every 5,000 iterations, comparing dropping the `Tree` vs explicit `tree.delete()`. External memory grows from ~69 MB to ~3.7 GB without delete and stays at ~66 MB with it. RSS tracks external memory, which is what you'd expect — the live JS heap is small, the leak lives in the Emscripten arena. This is convincing enough that I don't need a flame graph; the no-delete column is monotonically growing and the delete column is flat.

## Risks

The only correctness risk is using `tree.rootNode` after release. The diff keeps `rootNode.descendantsOfType(...)` strictly inside the scope, so that's fine. One thing I'd verify: does `collect` ever stash a reference to a node escaping the scope (e.g., in a closure inside the permission decision)? If `ask` enqueues a deferred decision that touches `rootNode` later, you'd get a use-after-free that surfaces as a WASM trap or, worse, silently wrong results. A short comment near the `acquireRelease` saying "rootNode references must not escape this scope" would harden the contract for future contributors.

The other thing worth a glance: error-path ordering. `Effect.acquireRelease` runs the release on interruption; if the run itself throws synchronously after the scope but before `run` is called, the release should still fire because `run` is outside the scope. That's the intent of moving `run` outside, but a smoke test that throws inside `ask` and asserts the release ran would be cheap to add.

## Verdict

**Verdict:** merge-after-nits

Nits: add an inline comment about node lifetimes, and consider one negative test that throws from inside `ask` and asserts cleanup. The fix itself is correct and the evidence is strong.

---

*Reviewed by drip-155, deadline tick.*
