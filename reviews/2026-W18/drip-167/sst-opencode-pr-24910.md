# sst/opencode PR #24910 — fix(tool): add 30s timeout to glob and grep to prevent filesystem hang

- Repo: sst/opencode
- PR: https://github.com/sst/opencode/pull/24910
- Head SHA: `207fd64fb95d862e02ada6dffbf544a000db70cb`
- Author: yelog (Logan Ye)
- Size: +85 / −10, 4 files
- Closes: #24830

## Context

When a model emits `glob({ path: "/" })` or `grep({ path: "/" })`, the
underlying ripgrep invocation can spin against the entire filesystem.
Background / autopilot loops where external-directory permission is
auto-accepted have no human in the loop to abort, so the tool turn hangs
forever and starves the session. The PR puts a 30-second hard ceiling on
both tools at `packages/opencode/src/tool/glob.ts` and
`packages/opencode/src/tool/grep.ts`, and turns the timeout into a
model-actionable message ("path too broad, use a more specific path") rather
than a failure.

## How the timeout is wired

The pattern is identical in both files. A new constant
`SEARCH_TIMEOUT_MS = 30_000` is declared at the top, and inside the
Effect-gen the existing `ctx.abort` is composed with
`AbortSignal.timeout(SEARCH_TIMEOUT_MS)` via `AbortSignal.any([...])`. The
combined signal is passed to `rg.files({ ..., signal })` /
`rg.search({ ..., signal })` so ripgrep is killed by the existing
`raceAbort` plumbing once either source fires. A `let timedOut = false`
flag is closed over and set inside an `Effect.catchIf` whose predicate is
the discriminator that matters here:

```
() => timeoutSignal.aborted && !ctx.abort.aborted
```

That check is the right one — it catches **only** the internal-deadline case
and lets a genuine user/session abort propagate as a real failure. Without
this discriminator the catch would swallow user-initiated cancels and
return a misleading "search too broad" message.

The output paths differ slightly:

- `glob.ts:72-95` returns an empty file list and prepends a one-line
  diagnostic: `Search timed out after 30s. The search path "<search>" is
  too broad. Use a more specific path closer to the target files.`
- `grep.ts:65-90` short-circuits with a synthesized result-shape
  `{ title, metadata: { matches: 0, truncated: false }, output: "..." }`
  and the call site checks `if ("output" in result) return result;` to
  bypass the normal row-formatting path.

Both messages are model-facing and actionable, which is the right framing —
the model retries with a narrower path on the next turn instead of treating
the hang as a tool failure.

## Things I'd want tightened

1. **Test asserts the wrong thing.** The two new tests at
   `glob.test.ts:81-99` and `grep.test.ts:113-131` fire `controller.abort()`
   immediately and then assert `Exit.isFailure(exit) === true`. That
   exercises the **user-abort** path, not the timeout path. The timeout
   logic is the actual new code, and the discriminating predicate
   (`timeoutSignal.aborted && !ctx.abort.aborted`) is what most needs a
   regression test — otherwise a refactor that flips the predicate to
   catch user aborts too will pass CI silently. A test with a slow fake
   `rg` that takes longer than a small `SEARCH_TIMEOUT_MS` (injected via
   constant or parameter) would exercise the actual fix.

2. **`SEARCH_TIMEOUT_MS` is a hardcoded constant.** Hot-fixing the timeout
   in production currently requires a code change. An env var
   (`OPENCODE_SEARCH_TIMEOUT_MS`) or a config value, with the 30s default
   preserved, would let operators bump it for legitimately huge
   monorepos without forking.

3. **Two near-identical implementations.** The timeout-signal composition,
   the catch predicate, and the diagnostic string are duplicated across
   `glob.ts` and `grep.ts`. A small helper in `./tool` (e.g.
   `withSearchTimeout(ctx, timeoutMs)` returning `{ signal, didTimeOut() }`
   plus a `formatTimeoutMessage(search, ms)` helper) would prevent drift —
   especially the discriminator predicate, which is the most subtle line and
   the easiest one to silently break in a refactor.

4. **`grep.ts` shape divergence is worth a comment.** The `if ("output" in
   result) return result;` short-circuit at the call site relies on the
   normal `rg.search` return shape *not* having a top-level `output`
   property. That's true today, but a one-line comment explaining the
   structural-typing trick prevents a future contributor from adding
   `output` to the normal result and silently bypassing all grep
   formatting.

5. **Zero-line file in glob.ts on timeout.** When `timedOut` is true, the
   output starts with the diagnostic and then includes nothing else —
   good. But `if (files.length === 0)` only triggers on the
   non-timed-out, no-results path; verify the model doesn't get a confusing
   "No files found" *plus* the timeout message because of an `&&`/`||`
   misorder. The current code uses `if (timedOut) {...} else if (files.length
   === 0) {...}` so it's fine, but the symmetry with `grep.ts` is worth
   asserting in a test.

## Suggestions

- Replace the abort-immediately tests with timeout-fires tests using a slow
  fake `rg` (or expose `SEARCH_TIMEOUT_MS` as a parameter for tests to
  override).
- Make `SEARCH_TIMEOUT_MS` env-overridable.
- Extract `withSearchTimeout` + `formatTimeoutMessage` helpers to dedupe
  glob and grep.
- Add a one-line comment at the `"output" in result` check in `grep.ts`
  explaining the structural-typing discriminator.

## Verdict

**merge-after-nits** — solid pattern, the discriminator predicate is right,
and the model-facing diagnostic is the correct framing. The test coverage
asserts a different path than the one the PR adds, which is the most
important nit; the rest are polish.
