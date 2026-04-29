# openai/codex PR #20100 — Increase plugin hook env test timeout

- **PR**: https://github.com/openai/codex/pull/20100
- **Author**: abhinav-oai
- **Merged**: 2026-04-29T00:08:12Z
- **Head SHA**: `94f3287be4e8`
- **Size**: +1/-1 across 1 file (`codex-rs/hooks/src/engine/mod_tests.rs`)
- **Verdict**: `merge-after-nits`

## Context

This is the follow-up companion to #20088 ("Fix flaky plugin hook env
test"). After #20088 stabilized the test's structural race, the test still
sporadically failed in CI under load because its inline `python3 …` hook
script — which spawns a fresh interpreter, imports `json`, dumps a small
payload, and exits — was being killed by the hook engine's per-handler
`timeout_sec` budget (5 s) before the cold-start interpreter finished. The
fix bumps that budget from 5 to 10 seconds.

## What changed

Single-line bump at `codex-rs/hooks/src/engine/mod_tests.rs:376`:

```diff
                 hooks: vec![HookHandlerConfig::Command {
                     command: format!("python3 {}", script_path.display()),
-                    timeout_sec: Some(5),
+                    timeout_sec: Some(10),
                     r#async: false,
                     status_message: None,
                 }],
```

The script itself is the same `print(json.dumps({...}))` one-liner from the
surrounding test fixture (visible in the unchanged context at line ~373:
`print(json.dumps({`).

## Why `merge-after-nits`

This is a defensible CI-stability fix and the right family of fix (extend
the budget for the slow path) rather than the wrong family ("retry
silently"). Concerns:

- **Doubling without measurement leaves the next bump unprincipled.** The
  PR doesn't reference the observed p99 startup time of `python3 -c
  "import json; …"` on the CI agent, so 10 s is an educated guess. If the
  test flakes again on a slower runner, the next person will face the
  same "what's the right number" question. A one-line comment near the
  literal — `// 5s was tight on cold-start CI agents (~3-4s observed);
  doubled with margin` — would make the next bump principled.

- **Diverges silently from any sibling timeouts.** This file likely has
  other inline-Python hook tests with the same 5 s budget; they'll still
  flake under the same conditions. Worth a 10-second `grep timeout_sec
  codex-rs/hooks/src/engine/mod_tests.rs` to either bump them in the same
  PR (and explain the unified bound) or to confirm only this one is
  load-bearing on the cold-start.

- **Hides rather than diagnoses the cold-start cost.** The deeper fix is
  often "don't spawn `python3` per test" — a one-shot subprocess pool,
  or `sys.executable` re-use, or a precompiled `.pyc` — but those are out
  of scope for a flake-killer PR and are appropriately deferred. Just
  worth filing a follow-up issue so the timeout doesn't keep creeping up.

## What I learned

CI-flake fixes split cleanly into two families: (a) "the test had a real
race, fix the race" (#20088 was that family) and (b) "the test budget was
tight against the real-world cost distribution, widen the budget" (this
PR). Family (b) is correct *as long as* you've already applied family (a)
or have ruled it out — otherwise you're hiding a structural race behind a
larger budget. The author got the order right by landing #20088 first and
then this. The reviewing-eye improvement is to make the budget number
explainable: "5s → 10s because the observed p99 cold-start is N s on
runner R" beats "5s → 10s because flaky" every time, even though both end
up at the same line.
