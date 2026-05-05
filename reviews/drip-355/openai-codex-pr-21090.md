# openai/codex PR #21090 — Dedupe fallback model metadata warnings

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/21090
- Head SHA: `6a4e364db319`
- Size: +69 / -16 across 5 files (core session/turn_context, state, tests; models-manager + tests)
- Fixes upstream #21070.

## Summary

Two small changes wearing one hat. (1) Dedupe the fallback-metadata
warning so each unresolved model slug only warns once per session
instead of every turn. (2) Loosen the namespaced-suffix lookup to allow
hyphens in the namespace segment, so provider IDs like `openai-codex/`
or `vertex-ai/` actually match the prefix-strip path. Both are
follow-ups on a user report (issue #21070).

## What I like

- The dedupe lives in
  `codex-rs/core/src/session/turn_context.rs:751-775`. The new control
  flow is:
  ```
  if !tc.model_info.used_fallback_model_metadata { return; }
  let should_emit = {
      let mut state = self.state.lock().await;
      state.fallback_model_metadata_warning_slugs.insert(tc.model_info.slug.clone())
  };
  if !should_emit { return; }
  self.send_event(... Warning ...).await;
  ```
  `HashSet::insert` returns `true` only if the value was new, so the
  short-circuit is one atomic check-and-set under the lock. No
  TOCTOU.
- The lock is **released before `send_event`** (note the `let
  should_emit = { ... }` block scopes the `MutexGuard`). That's the
  correct shape for "emit-an-event-while-holding-state-lock"
  patterns — sending an event can be `.await`-y and you don't want to
  hold the session lock across it. Easy to get wrong.
- `SessionState::fallback_model_metadata_warning_slugs: HashSet<String>`
  is added to both the struct definition and `SessionState::new()`
  (`codex-rs/core/src/state/session.rs:27,51`). Matches the pattern of
  the neighboring `mcp_dependency_prompted: HashSet<String>`, so this
  is consistent with how related "warn-once" state is already kept.
- Test at `codex-rs/core/src/session/tests.rs:6016-6038`:
  call `maybe_emit_unknown_model_warning_for_turn` twice with the
  same slug, assert the first event arrives with the expected message
  text, assert the second `try_recv()` returns `Err`. Exact-string
  match on the warning body locks the user-visible copy.
- The hyphen fix at `codex-rs/models-manager/src/manager.rs:431-435`:
  `c.is_ascii_alphanumeric() || c == '_' || c == '-'` plus a
  `namespace.is_empty()` guard. The empty-namespace guard is there
  because adding `'-'` to the allowed set means the all-empty string
  technically passed `.chars().all(...)` (vacuous truth). Catching
  that explicitly is the right thing.
- The new test
  `get_model_info_matches_hyphenated_provider_namespace_suffix` at
  `manager_tests.rs:298-311` uses `openai-codex/gpt-image` as the
  namespaced slug and asserts `used_fallback_model_metadata == false`
  on the lookup result. Concrete and focused.

## Nits / discussion

1. **Warning text is locked into a test.** That's good for stability,
   bad if anyone tries to refactor the message and forgets the test
   string. The body is duplicated verbatim in
   `turn_context.rs:768-771` and `tests.rs:6028-6031`. A `const
   FALLBACK_METADATA_WARNING_FMT: &str = "Model metadata for `{}` …"`
   shared between both would be marginally more refactor-safe.

2. **Cross-session dedupe vs cross-turn dedupe.** The dedupe is keyed
   on the live `SessionState` instance, so a fresh session re-warns —
   probably the desired behavior, but the PR description says "warn
   once per session" without saying that explicitly. Worth confirming
   in the changelog so users don't expect "warn once globally / until
   I clear cache".

3. **Hyphen widening is broader than it looks.** `is_ascii_alphanumeric
   || '_' || '-'` now matches things like `-leading-hyphen/model` or
   `trailing-/model`. The empty-namespace guard catches `""`, but a
   leading-hyphen namespace still passes. Probably benign (the lookup
   table would just not have an entry), but if there's any place that
   uses namespace as a directory name on disk, leading hyphens become
   POSIX option-parsing footguns. Worth a stricter regex like
   `^[a-z0-9_][a-z0-9_-]*$` if you want to be safe.

4. **Two unrelated fixes in one PR.** The dedupe and the hyphen fix
   share an issue but not really a code path. Splitting would have
   given each its own bisectable commit; not blocking.

## Verdict

**merge-after-nits.** Both halves are correct and well-tested. The
lock-release-before-send pattern is the right shape, the empty-
namespace guard is a nice catch on the hyphen widening, and the
exact-string assertion on the warning body keeps the user-facing
copy stable. Worth tightening the namespace pattern (no leading
hyphens) before merge if you want to fully close the
"weird-namespace-as-path" surface.
