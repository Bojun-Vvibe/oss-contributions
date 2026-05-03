# QwenLM/qwen-code PR #3714 — feat(core): write runtime.json sidecar for active sessions

- **Repo:** QwenLM/qwen-code
- **PR:** #3714
- **Head SHA:** `e1e8d29be452cb3422f0156358af873fdba336fa`
- **Author:** yeelam-gordon
- **Title:** feat(core): write runtime.json sidecar for active sessions
- **Diff size:** +613 / -0 across 6 files
- **Drip:** drip-292

## Files changed

- `packages/core/src/utils/runtimeStatus.ts` (+250/-0) — new module that writes/refreshes/cleans a `runtime.json` sidecar per session.
- `packages/core/src/utils/runtimeStatus.test.ts` (+271/-0) — comprehensive unit coverage for write, stale detection, cleanup.
- `packages/core/src/config/config.ts` (+56/-0) — wires sidecar write/refresh into config lifecycle.
- `packages/core/src/config/storage.ts` (+12/-0) — adds path helpers for the sidecar location.
- `packages/cli/src/gemini.tsx` (+23/-0) — installs the sidecar writer + cleanup on process exit signals.
- `packages/core/src/index.ts` (+1/-0) — re-exports the new utility.

## Specific observations

- This PR is **pure addition** (+613 / -0) — no existing behaviour changes. The risk surface is bounded to "does the new sidecar leak files or block startup".
- `runtimeStatus.test.ts` (+271 lines) is larger than `runtimeStatus.ts` itself (+250). That's a healthy ratio for a feature whose correctness story is "files appear and disappear at the right times".
- The PR body explicitly cites prior art in peer interactive CLIs and frames the contract as "PID → session id + cwd mapping for external tools". That's the right framing — without an external-tooling contract this would be over-engineering.
- `config.ts` (+56) and `gemini.tsx` (+23) are the two integration touchpoints. Reviewers should specifically check:
  - Does the sidecar write happen *after* the chat-log directory exists, or does it race? (The diff cap prevented inspecting the actual call site.)
  - Is the cleanup wired to *all* exit paths — `SIGINT`, `SIGTERM`, normal exit, uncaught exception? A leaked `runtime.json` after an unclean exit is the obvious failure mode and the test file should cover stale-cleanup at next startup.
- `storage.ts` (+12) adding path helpers is the right place; keeps `runtimeStatus.ts` decoupled from the storage layout.
- `index.ts` (+1) exports the helper publicly. Confirm whether external consumers should depend on this directly or only read the sidecar file — exporting the writer API to `core/index` invites third-party plugins to write *their own* sidecars, which may not be intended.
- No banned-string / secret risk: this is local filesystem state, no network surface.
- One latent concern not visible in the diff: cross-process safety. If two CLI instances start in the same session directory (rare but possible with concurrent windows), do they trample each other's sidecar? An atomic write (`writeFile` to tmp + `rename`) plus a PID-prefixed filename would side-step this entirely.

## Verdict: `merge-after-nits`

Tightly scoped, well-tested, additive feature with a clear external contract. Asks before merge: (1) confirm cleanup fires on every exit path including uncaught exceptions, (2) confirm atomic write semantics if two CLI processes can share a session dir, and (3) decide whether `runtimeStatus` should be public API (`core/index`) or internal. None of these block conceptually; they're all "make sure the corner cases are explicit".
