# openai/codex#20520 — Persist selected environments in turn context replay

- **PR**: https://github.com/openai/codex/pull/20520
- **Head SHA**: `97e1ace5b90fe25ebd8258e5b72a1320a8957a8c`
- **Verdict**: `merge-after-nits`

## What it does

Persists `TurnEnvironmentSelection`s into the `TurnContextItem` rollout
record, then teaches resume/fork/replay paths to reconstruct the prior
selection rather than always defaulting back to "local environment for
config.cwd." +149 / -27 across 12 files.

## What's load-bearing

- `codex-rs/core/src/session/turn_context.rs:331-340` — the *write* side:
  `TurnContext::to_rollout_item()` now sets
  ```
  environments: (!self.environments.is_empty()).then(|| {
      self.environments.iter().map(TurnEnvironment::selection).collect()
  }),
  ```
  The `(!is_empty()).then(|| ...)` shape means the field stays `None` for
  threads that never selected explicit environments, preserving the prior
  on-disk format for the common case (no rollout-format churn for users
  who don't use environments).
- `codex-rs/protocol/src/protocol.rs` adds `environments:
  Option<Vec<TurnEnvironmentSelection>>` to `TurnContextItem`. Optional
  field with serde default keeps deserialization backward-compatible with
  existing rollout files written before this PR — important because the
  feature touches the on-disk replay format.
- `codex-rs/core/src/thread_manager.rs:1381-1395` — the *read* side:
  ```
  fn restore_thread_environment_selections_from_history(
      history: &InitialHistory,
  ) -> Option<Vec<TurnEnvironmentSelection>> {
      history.get_rollout_items().iter().rev().find_map(|item| match item {
          RolloutItem::TurnContext(context) => context
              .environments
              .clone()
              .filter(|environments| !environments.is_empty()),
          _ => None,
      })
  }
  ```
  Walks rollout items in reverse and picks the most recent
  `TurnContext` that recorded a non-empty selection — this is the right
  semantic ("last selection wins on resume"), and the `.filter(|envs|
  !envs.is_empty())` guards against empty-Vec records.
- `codex-rs/core/src/thread_manager.rs:613-823` — four call sites are
  flipped from `default_thread_environment_selections(...)` to
  `restore_..._from_history(&initial_history).unwrap_or_else(|| default_...)`:
  start-thread (`:613-623`), resume-thread (`:668-679`), fork-thread
  (`:810-823`), and the inner spawn-thread state path (`:962-980`). The
  consistency of the pattern (`.or_else(restore).unwrap_or_else(default)`)
  across all four sites is the load-bearing discipline that prevents
  drift.
- `codex-rs/core/src/thread_manager_tests.rs:381-444` — the test name
  flips from `..._do_not_restore_thread_environments_from_rollout` to
  `..._restore_thread_environments_from_rollout`, and the new body uses
  the actual rollout-recorder roundtrip (writes via session, reads via
  `RolloutRecorder::get_rollout_history`, asserts the persisted
  `environments` field) — exercising the on-disk format end-to-end rather
  than just the in-memory transformer. Right shape.
- All `..._tests.rs` and `tests/suite/*.rs` updates add `environments:
  None,` to `TurnContextItem` constructions (mechanical; pinned by `cargo
  build` failing without them since the field is non-default).

## Concerns

1. **PR body says "Testing: not run."** This PR touches the on-disk
   rollout format for every codex thread. At minimum
   `cargo test -p codex-core thread_manager_tests` and
   `cargo test -p codex-core rollout_reconstruction_tests` should be run
   locally; the new mechanical `environments: None,` insertions across 12
   call sites are easy to get wrong by one site. Please run the affected
   suites before merge and update the PR body.
2. **Migration story for in-flight rollouts is implicit.** A user with
   active rollouts written *before* this PR will have `environments:
   None` for all historical TurnContext entries, so resume will fall
   through to `default_thread_environment_selections(config.cwd)` — which
   is the prior behavior. That's correct (and the optional-field design
   supports it), but it should be stated in the PR body so reviewers
   don't have to derive it.
3. **`restore_..._from_history` reads the *most recent* non-empty
   selection. A thread that explicitly switched from environment-A to
   environment-B mid-thread, then back to default (empty), would resume
   into environment-B.** The `.filter(|envs| !envs.is_empty())` skips
   empty selections, so a deliberate "I unselected all environments"
   intent gets lost. Probably acceptable, but worth a comment at
   `:1381` either confirming this is intentional or noting it as
   follow-up.
4. **No test for the "history has multiple TurnContext entries with
   different environment selections" case.** The new `thread_manager`
   test seeds one source turn. A second test asserting "TurnContext-A
   with envs=[X], then TurnContext-B with envs=[Y] → resume reads [Y]"
   would pin the reverse-walk semantics directly.

## Verdict

`merge-after-nits` — the design is correct (optional field for back-compat,
write side narrowly scoped to non-empty, read side walks reverse for
"last wins"), and the consistency across four call sites is the right
discipline. Block on running the test suites (#1) before merge; #2-#4 are
follow-up polish.
