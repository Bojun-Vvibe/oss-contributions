# google-gemini/gemini-cli PR #25999 — `feat`: bypass browser auth in yolo mode for browser-less environments

- **PR**: https://github.com/google-gemini/gemini-cli/pull/25999
- **Author**: @renuka16032007
- **Head SHA**: `b17f34f6`
- **Size**: +10 / −1 (2 files)
- **Files**: `packages/cli/src/acp/acpClient.ts`, `packages/cli/src/utils/resolvePath.ts`

## Summary

Aims to fix #25872 (CLI hangs in browser-less environments like Termux when a tool requires browser-based approval, even when `yolo` mode is active). The intended behavior is reasonable: in yolo mode, auto-allow the request that would otherwise wait on a browser signal. The PR also bundles a second, unrelated change to `resolvePath.ts` to swallow `ENAMETOOLONG` errors from `path.normalize`.

## Verdict: `request-changes`

Both changes have correctness problems that block merge as written. The `acpClient.ts` change is a syntax error — the `if` is interpolated into the middle of an object literal and will not compile. The `resolvePath.ts` change has stray indentation that breaks the existing prettier style and is an unrelated concern that doesn't belong in the same PR.

## Specific references

- `packages/cli/src/acp/acpClient.ts:817-823` — the diff inserts the early-return statement *inside an object literal*:
  ```diff
                  sessionUpdate: part.thought
                    ? 'agent_thought_chunk'
                    : 'agent_message_chunk',
  +    if (this.config.yolo) { return { allowed: true }; }
                  content,
                });
  ```
  This is **not valid TypeScript**. The surrounding code is constructing an object passed to `tx.send(...)` (or similar) — `sessionUpdate: ...,` is an object property, and an `if` statement cannot appear between two property declarations. The code as written will fail `tsc` immediately. The author's intent (auto-allow when `this.config.yolo`) is sound, but the placement is wrong: the early return belongs at the top of the *enclosing function* that handles the permission request, not inside the object literal that emits a session-update chunk. Looking at the context, this code is in the streaming-update path (it's branching on `part.thought` for the `agent_thought_chunk` vs `agent_message_chunk` discrimination), which is the wrong code path entirely — session updates aren't permission requests. The yolo-bypass needs to be in whichever method handles the permission *prompt* (likely `requestPermission` or similar), not in the streaming chunk emitter.
- `packages/cli/src/utils/resolvePath.ts:19-29` — the second change wraps `path.normalize` in a try/catch that swallows `ENAMETOOLONG`:
  ```typescript
        try {
        return path.normalize(expandedPath);
      } catch (err: any) {
        if (err.code === 'ENAMETOOLONG') {
          return expandedPath;
        }
        throw err;
        }

  ```
  Three problems: (a) indentation is broken (mixed 4-space and 6-space), the closing `}` of the try has odd alignment, and there are stray blank lines after — this won't pass prettier; (b) the change is unrelated to the yolo bypass and silently expands the PR's blast radius (a maintainer reviewing "yolo bypass" doesn't expect to also be reviewing path-resolution semantics); (c) the rationale isn't explained anywhere — *why* does `ENAMETOOLONG` need to be swallowed and the un-normalized path returned? Returning a non-normalized path from a function called `resolvePath` violates its contract. The honest fix for "user passed a too-long path" is to throw a wrapped error (`PathTooLongError`) that callers can render as a user-friendly message, not to silently return a possibly-broken value.

## What needs to change

1. **Move the yolo-bypass to the actual permission-request handler.** Search for the method that handles the ACP permission prompt (probably named something like `requestPermission`, `confirmExecution`, `awaitUserApproval`, or similar) and add the early return at the top:
   ```typescript
   async requestPermission(...): Promise<{ allowed: boolean }> {
     if (this.config.yolo) {
       return { allowed: true };
     }
     // existing browser-flow code
   }
   ```
   The PR description says the user message will be `"Since you have yolo mode enabled, the request was automatically allowed."` — that string isn't in the diff. It needs to actually be emitted (probably via the same info/log channel the existing browser flow uses for status messages) so users in browser-less environments understand why a prompt didn't appear.
2. **Drop the `resolvePath.ts` change from this PR**, file it as a separate issue, and write up the actual `ENAMETOOLONG` reproduction. If the path is genuinely too long for the OS, returning the un-normalized path doesn't help anyone — the next syscall will fail with `ENAMETOOLONG` anyway.
3. **Add a test** that constructs a `Session` with `config.yolo = true`, invokes the permission-request method, and asserts the returned `{ allowed: true }` shape with no browser interaction. Without a test, this regression door reopens the next time someone refactors the permission flow.
4. **Verify on a real browser-less environment** (the PR description's testing matrix has every cell ticked including Termux/headless servers, but the code as written wouldn't compile, so the testing claims need re-validation after the fix).

## What I learned

A category of bugs that's invisible to syntax-checking-only review: *placing valid code at an invalid syntactic position in a diff*. The line `if (this.config.yolo) { return { allowed: true }; }` is well-formed TypeScript on its own, and a reviewer skimming the diff in a web UI may visually parse the *intent* without catching that the surrounding context is an object literal that disallows statements. The defense is `tsc --noEmit` in CI on every PR, plus a habit of always reading 5-10 lines of diff *context* (not just the `+` lines) before approving anything. Separately, this PR illustrates why "while I'm in there" changes (the `resolvePath` edit) hurt PR throughput — they double the review surface and entangle two independent decisions, often delaying both.
