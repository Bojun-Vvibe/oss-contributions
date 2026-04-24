# openai/codex PR #19454 — Split approval matrix test groups

- **Repo:** openai/codex
- **PR:** [#19454](https://github.com/openai/codex/pull/19454)
- **Head SHA:** `bb482f08c09931aec1dd7d7703a484f10715c1a1`
- **Author:** dylan-hurd-oai (Dylan Hurd)
- **Size:** +63 / −3 across 1 file
- **Reviewer:** Bojun (drip-26)

## Summary

Companion to drip-26's other dylan-hurd-oai PR (#19452) — same goal
(stabilise CI), different mechanism. The single
`approval_matrix_covers_all_modes` test in
`codex-rs/core/tests/suite/approvals.rs` packed every approval /
sandbox / tool scenario combination into one `#[tokio::test]` body
and was repeatedly hitting the 60-second per-test Linux remote
timeout (7 cited CI runs in the PR body).

The fix splits that monolith into five focused tests by scenario
group, each running a filtered subset of the same shared scenario
table. Coverage stays in the same `scenarios()` constructor; only
the test-runner partitioning changes.

## Key changes (single file)

### `codex-rs/core/tests/suite/approvals.rs:574–580` — new enum

```rust
enum ScenarioGroup {
    DangerFullAccess,
    ReadOnly,
    WorkspaceWrite,
    ApplyPatch,
    UnifiedExec,
}
```

One per logical slice of the matrix.

### `codex-rs/core/tests/suite/approvals.rs:1673–1696` — five test wrappers

Each is a one-line `#[tokio::test(flavor = "multi_thread", worker_threads = 2)]`
that calls `run_scenario_group(<group>)`. No behavioural divergence
between them — same harness, different filter.

### `codex-rs/core/tests/suite/approvals.rs:1698–1717` — `run_scenario_group`

```rust
let scenarios = scenarios()
    .into_iter()
    .filter(|scenario| scenario_group(scenario) == group)
    .collect::<Vec<_>>();
assert!(!scenarios.is_empty(), "expected scenarios for {group:?}");

for scenario in scenarios {
    run_scenario(&scenario)
        .await
        .with_context(|| format!("approval scenario failed: {}", scenario.name))?;
}
```

Two nice details:

- The `assert!(!scenarios.is_empty(), ...)` is a guard against a
  future scenario being added that doesn't fit any group — the test
  for that group will fail loudly instead of silently passing with
  zero coverage.
- The `with_context(|| format!("approval scenario failed: {}",
  scenario.name))` improves the diagnostic when a single scenario
  inside a group fails — the previous monolith reported only "test
  X timed out" without telling you which scenario was slow. Worth
  the new `use anyhow::Context;` import on line 3.

### `codex-rs/core/tests/suite/approvals.rs:1719–1735` — `scenario_group` classifier

The classifier is action-kind-first, sandbox-policy-second, with
`ApplyPatch` and `UnifiedExec` taking precedence regardless of
sandbox. That matches the operational reality — apply-patch and
unified-exec have their own approval flows that are independent
of the sandbox-policy axis.

`SandboxPolicy::ExternalSandbox` is folded into the
`WorkspaceWrite` group, which is a defensible choice: external
sandboxes share the workspace-write enforcement path. Worth a
one-line comment noting that mapping is intentional, because the
next reader will assume it's a missing `ExternalSandbox` group.

## What's good

- Pure test-side change. No production code modified. Zero risk
  to runtime behaviour.
- The `scenarios()` constructor is reused untouched — no chance
  of accidentally dropping coverage during the split.
- The `assert!(!scenarios.is_empty(), ...)` future-proofs the
  partitioning against silent coverage loss when a new
  `ActionKind` or `SandboxPolicy` variant lands without a
  classifier update. (Though see concern #2 below.)
- `with_context` on the inner `run_scenario` call materially
  improves debuggability — previously a 60s timeout told you
  nothing; now it tells you the scenario name that was
  in-flight.

## Concerns

1. **Classifier exhaustiveness is structural, not enforced.** The
   `match` on `scenario.action` enumerates `ApplyPatchFunction`,
   `ApplyPatchShell`, `RunUnifiedExecCommand`, and a four-variant
   `WriteFile | FetchUrlNoProxy | FetchUrl | RunCommand` arm. If a
   new `ActionKind` variant is added without updating
   `scenario_group`, the compiler will catch it (no `_` arm). Good.
   But the inner `match &scenario.sandbox_policy` *also* has no
   `_` arm and currently has four explicit variants — same
   property, also good. Worth confirming there's no `#[non_exhaustive]`
   on either enum that would weaken this. (The diff doesn't show
   the enum definitions; this is a confirm-on-merge item.)

2. **`assert!(!scenarios.is_empty())` only catches "all five groups
   non-empty at the time the test was written"-class mistakes.** If
   someone deletes the last scenario for a group, the test will
   suddenly fail with "expected scenarios for ApplyPatch", which
   may or may not be the desired signal — they may have intentionally
   removed apply-patch scenarios. Consider downgrading to a
   `eprintln!` warning, or document that "deleting the last
   scenario in a group requires also deleting the group test".

3. **`worker_threads = 2` × 5 tests = up to 10 concurrent worker
   threads** if the test runner schedules them in parallel. The
   monolith was 1 × 2 = 2. On a constrained CI runner this may
   trade per-test timeout pressure for total-CPU pressure. Probably
   fine because the per-test timeout is the hard limit being
   addressed, but worth measuring on the same runners that were
   timing out.

4. **Nit:** The five test functions are byte-identical except for
   the group enum value. A `paste::paste!`-style macro or a single
   parameterised test would reduce duplication, but the trade-off
   is worse cargo-test output ergonomics. The current form is fine
   for five entries; if it grows to ten, revisit.

## Risk

Low. Test-file-only. Coverage is preserved by construction (same
`scenarios()` source). Only meaningful regression risk is
classifier drift — addressed by the no-`_`-arm match.

## Verdict

**merge-as-is**

Solid CI-stability fix that improves diagnostics as a side effect.
The four concerns above are doc/follow-up nits, not blockers.

## What I learned

When a parameterised test starts hitting per-test timeouts in CI,
"split it by partition key" beats "raise the timeout" because the
new failure mode (one group's wall clock) is bounded by your
slowest *group*, not your slowest *individual scenario plus everyone
else after it*. The `with_context` wrapper around the inner
scenario runner is the cheapest possible way to recover the "which
scenario blew up" diagnostic that the split otherwise loses.
