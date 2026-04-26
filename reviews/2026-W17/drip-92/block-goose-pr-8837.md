---
pr: 8837
repo: block/goose
sha: 14aaee8f6e90799337327429a7259250a6e85d58
verdict: merge-as-is
date: 2026-04-27
---

# block/goose #8837 — fix(ci): prevent flaky smoke test timeouts from failing the build

- **Author**: michaelneale
- **Head SHA**: 14aaee8f6e90799337327429a7259250a6e85d58
- **Size**: +36/-11 in `ui/desktop/tests/integration/test_providers_lib.ts`.

## Scope

Vitest's per-test `testTimeout` (60s default) **aborts tests externally** — the resulting error is not catchable via try/catch. Flaky-provider smoke tests (`nvidia/nemotron-3-nano-30b-a3b` is the cited example) sometimes hang past 60s, and the existing flaky-test wrapper's try/catch can't suppress vitest's external timeout. Fix: add an internal 55s timeout to `runGoose()` (catchable, fires inside the promise) and bump vitest's per-test timeout to 90s for flaky tests so the internal one always trips first.

## Specific findings

- `test_providers_lib.ts:316-328` — `test.each(flaky)` now passes `90_000` as the third arg to give vitest's external timeout a 35s margin over the internal one. Standard vitest API.
- `test_providers_lib.ts:357-368` — `runGoose()` signature gains `timeoutMs: number = 55_000`. Default-arg, back-compat for any other caller.
- `test_providers_lib.ts:380-389` — internal timer + `settled` flag. `setTimeout` fires `child.kill('SIGKILL')` and `reject(new Error(\`goose timed out after ${timeoutMs}ms\\n\\nPartial output:\\n${output}\`))`. Including the partial output in the error message is a nice touch — the try/catch in the flaky wrapper at `:323` logs `err.toString()` so the partial output reaches the CI log for diagnostics. Good.
- `test_providers_lib.ts:399-419` — `child.on('close')` and `child.on('error')` both check `if (!settled)` before resolving and clearing the timer. Standard guard; without it, a process that errors *and* closes could double-resolve / leak the timer. Correct.
- The previous code path `child.on('error', err => resolve(\`spawn error: ${err.message}\`))` is preserved (resolves rather than rejects on spawn errors), now timer-aware. Note the asymmetry: spawn errors → `resolve("spawn error: …")`, timeouts → `reject(Error)`. The flaky-test wrapper at `:319-325` uses try/catch so it catches the timeout reject, but a spawn error would resolve through to `await fn(tc)` and only fail downstream when assertions run on the "spawn error: …" string. Same as pre-fix behaviour, not regressed.
- The 55s/90s split has 35s of headroom for the SIGKILL-then-resolve to actually drain, which is generous and won't be the cause of CI flakes itself.
- No test for the timeout path itself — but this *is* the test infra, so testing the test infra would be turtle-tower territory. Reasonable to skip.

## Risk

Negligible. Touches only the integration test infrastructure, doesn't ship to runtime. SIGKILL on a `goose` subprocess is the right escalation (the child has its own internal cleanup hooks but they're not blocking on shutdown for a smoke test). Worst case: slightly longer CI runs for genuinely-hanging providers (55s vs the previous 60s vitest abort).

## Verdict

**merge-as-is** — surgical fix to a real CI flake mechanism (vitest external timeouts being uncatchable), with proper double-resolve guards via the `settled` flag and partial-output preserved in the rejection message. The `timeoutMs` default-arg keeps the change non-breaking for other callers.
