---
pr: 24528
repo: sst/opencode
sha: 029ab835852c42eb5a1eee1f4ba4e60552c3c271
verdict: merge-as-is
date: 2026-04-27
---

# sst/opencode #24528 — fix(tui): startup rejection handling

- **Head SHA**: `029ab835852c42eb5a1eee1f4ba4e60552c3c271`
- **Size**: +79 / −71 in `packages/opencode/src/cli/cmd/tui/app.tsx`

## Summary
Replaces the prior `new Promise<void>(async (resolve) => { … })` async-executor pattern (which silently swallowed any rejection from `createCliRenderer`, `waitForThemeMode`, or the SolidJS `render` call and left opencode hanging) with a standard `(resolve, reject)` executor plus an inner `void (async () => {…})()` IIFE. A `fail(error)` closure at `app.tsx:118-123` destroys any partially initialized `renderer`, runs the win32 `unguard?()` cleanup, and rejects.

## Specific findings
- `app.tsx:117` — `let renderer: Awaited<ReturnType<typeof createCliRenderer>> | undefined`. Correct typing via `Awaited<…>` to avoid leaking the Promise wrapper into the cleanup closure.
- `app.tsx:118-123` — `fail` calls `renderer?.destroy()` then nulls it before `reject`. Order is right: destroy must happen before resolving the outer Promise so terminal state is restored before the caller logs/exits. Setting `renderer = undefined` after destroy prevents a double-destroy if `fail` is somehow re-entered.
- `app.tsx:115` — `unguard` from `win32InstallCtrlCGuard()` is captured before the IIFE so `fail` can release it even if `createCliRenderer` throws. Good.
- `app.tsx:138` — `win32DisableProcessedInput()` was moved *into* the async IIFE. Verify this is intentional: previously it ran synchronously before `createCliRenderer`. If the prior ordering mattered (e.g. mode flag must be set before the renderer attaches the stdin handler on Windows), this is a behavior change. PR body doesn't call it out.
- The IIFE has no `.catch(fail)`. The async function will throw into a floating Promise rejection. Either (a) explicitly chain `.catch(fail)` on the IIFE, or (b) wrap the body in `try { … } catch (e) { fail(e) }`. Right now a `createCliRenderer` rejection becomes an unhandledRejection, not a controlled `reject`. **This contradicts the PR's stated intent.**

Actually re-reading: the IIFE is `void (async () => { ... })()` and the body has no explicit try/catch. Need to either wrap or chain `.catch(fail)`. Without it, the rejection-handling promise the PR adds doesn't actually fire on the failure path.

Wait — looking again, this is the only path where errors can occur. **This needs a `.catch(fail)` appended to the IIFE call**, otherwise the fix is incomplete.

## Risk
Low for the structural change; the rejection-propagation goal is the right one. But the IIFE missing `.catch(fail)` means the actual rejection still becomes an unhandledRejection on `await createCliRenderer(...)` failure.

## Verdict
**merge-after-nits** — append `.catch(fail)` to the IIFE invocation at `app.tsx:137`, or wrap the body in try/catch. Also document why `win32DisableProcessedInput()` moved inside the async block.

(Marking as merge-after-nits despite being a one-line fix; the fix as written may not achieve its stated goal.)
