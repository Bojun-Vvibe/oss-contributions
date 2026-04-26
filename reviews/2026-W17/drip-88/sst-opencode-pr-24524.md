---
pr: 24524
repo: sst/opencode
sha: 2837bd8a0e3a48686bd2693f5fd933787fafc7aa
verdict: merge-as-is
date: 2026-04-27
---

# sst/opencode #24524 — fix(tui): handle background sync rejection in sync.tsx

- **Author**: alfredocristofano
- **Head SHA**: 2837bd8a0e3a48686bd2693f5fd933787fafc7aa
- **Size**: +13/-5 in `packages/opencode/src/cli/cmd/tui/context/sync.tsx`.

## Scope

The TUI's background sync built a `Promise.all([...])` over ~12 SDK calls and discarded the resulting promise via `void Promise.all(...)`. If any of those promises rejected, the rejection was unhandled (Node's `unhandledRejection` warning at best, process-level error handling at worst), and the TUI's `store.status` was already flipped to `"complete"` only inside the `.then()` — so a partial failure left the status stuck on `"partial"` indefinitely. This PR drops `void`, returns the promise from the outer `.then()` callback so the chain is awaited, and adds a `.catch(e => { Log.Default.error(...); setStore('status', 'complete') })` so the TUI always reaches a terminal status.

## Specific findings

- `packages/opencode/src/cli/cmd/tui/context/sync.tsx:421-444` — `void Promise.all([...])` → `return Promise.all([...])`. Correct: by returning the promise from the `.then()` callback, the outer chain now awaits it, and the existing `.catch(async (e) => {...})` at the end of the file (the "tui bootstrap failed" handler) becomes reachable for inner rejections. Net behavior: rejections are observed exactly once.
- `packages/opencode/src/cli/cmd/tui/context/sync.tsx:441-449` — the new inner `.catch` logs `error.message`, `error.name`, `error.stack` (with `e instanceof Error` guards) before forcing `setStore("status", "complete")`. The status flip-on-failure is the user-visible win: the TUI stops looking like "still loading" when a single sync call (e.g., `provider.auth`, `vcs.get`) 500s. Logging shape matches the existing "tui bootstrap failed" handler's shape, so the operator-side log analysis stays uniform.
- The order of operations is right: log first, *then* flip status. If `setStore` itself somehow threw (it shouldn't, but), the error would still be logged.
- One subtle behavior delta worth calling out: previously the outer `.then(() => { setStore("status", "complete") })` only ran if every inner promise resolved. Now, on inner rejection, the inner `.catch` flips status to `"complete"` *and* the outer `.catch` (the bootstrap-failed handler) does NOT fire — the rejection is consumed by the inner `.catch`. That's the intended behavior (the bootstrap proper succeeded, just one of the parallel post-bootstrap fetches failed), but the inner-catch-swallows-from-outer-catch interaction deserves a one-line comment so the next reader doesn't try to "fix" by re-throwing.

## Risk

Very low. Pure error-handling correctness fix on a code path that was provably broken (unhandled rejection + stuck `"partial"` status). No new dependencies, no behavior change on the success path.

## Verdict

**merge-as-is** — the fix is right, minimal, and addresses a real user-visible bug class. A one-line comment about why the inner `.catch` deliberately doesn't re-throw would be nice to have but isn't required.
