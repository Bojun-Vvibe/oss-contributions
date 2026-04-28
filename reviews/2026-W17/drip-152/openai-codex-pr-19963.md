# openai/codex#19963 — feat: fix hinting 3

- **Repo:** [openai/codex](https://github.com/openai/codex)
- **PR:** [#19963](https://github.com/openai/codex/pull/19963)
- **Head SHA:** `087df79a4a9f61b0a71234b5ffb2386108c4437a`
- **Size:** +51 / -16 across 3 files (`agent/control.rs`, `session/mod.rs`, `session/tests.rs`)
- **State:** MERGED (follow-up to PR #19805 review thread `r3153265562`)

## Context

The MultiAgentV2 usage-hint texts (root-agent guidance + subagent
guidance) had a subtle bug in their feature-gating: the
`configured_multi_agent_v2_usage_hint_texts` method on `Codex`
checked `config.features.enabled(Feature::MultiAgentV2)` against the
*original* config snapshot (`session_configuration.original_config_do_not_use`)
rather than the *current effective* feature state on the session.
Result: if a session's effective feature flags diverged from the
config-time snapshot (which can happen when feature overrides are
applied at session-construction time, or when a feature is dynamically
enabled/disabled mid-session), the hint texts would be governed by the
stale snapshot — either suppressed when they should fire or fired when
they should be suppressed.

## Design analysis

The fix moves the function from `Codex` to `Session` and reads from
`self.features` (the live effective feature state) instead of the
original config:

**Before** at `session/mod.rs:764-779` (deleted):
```rust
impl Codex {
    pub(crate) async fn configured_multi_agent_v2_usage_hint_texts(&self) -> Vec<String> {
        let state = self.session.state.lock().await;
        let config = &state.session_configuration.original_config_do_not_use;
        if !config.features.enabled(Feature::MultiAgentV2) {
            return Vec::new();
        }
        [...].collect()
    }
}
```

**After** at `session/mod.rs:843-857`:
```rust
impl Session {
    pub(crate) async fn configured_multi_agent_v2_usage_hint_texts(&self) -> Vec<String> {
        if !self.features.enabled(Feature::MultiAgentV2) {
            return Vec::new();
        }
        let state = self.state.lock().await;
        let config = &state.session_configuration.original_config_do_not_use;
        [...].collect()
    }
}
```

Two structural improvements:

1. **Lock-after-check** — the new shape checks features *before*
   acquiring the state lock, where the old shape acquired the lock
   *then* checked the snapshot. Since the new check reads from
   `self.features` (which is presumably an `Arc` or similar
   lock-free shared state), the disabled-feature path now returns
   without ever taking the state mutex. Real perf win on hot
   call sites where the feature is off.

2. **Source-of-truth consolidation** — `self.features` is the *single*
   source of truth for runtime feature state on the session. The old
   call dipped into `original_config_do_not_use` (the name itself
   warning callers off it!) precisely because the function was on the
   wrong type — `Codex` doesn't carry `features`, only `Session` does.
   Moving the function follows the data.

The caller-site update at `agent/control.rs:403-407` adds the missing
`.session` indirection (`parent_thread.codex.configured_multi_agent_v2_usage_hint_texts()`
→ `parent_thread.codex.session.configured_multi_agent_v2_usage_hint_texts()`).
This is a one-character change but load-bearing — it's the call site
that now reads through to the Session-level (live) feature state.

## Test coverage

Two new tests at `session/tests.rs:5434-5466` directly pin the new
contract via the test-only `Arc::get_mut(&mut session)` mutation
escape hatch:

1. **`configured_multi_agent_v2_usage_hint_texts_use_effective_enabled_feature_state`**
   constructs a session with the feature *disabled* in config
   (`make_multi_agent_v2_usage_hint_test_session(false)`), then
   forces `effective_features` to include `MultiAgentV2` via
   `Arc::get_mut(...).features = effective_features.into()`, and
   asserts the hint texts are returned. This is the exact bug the
   PR fixes — old behavior would have returned `Vec::new()` because
   the *config* snapshot still says disabled.

2. **`configured_multi_agent_v2_usage_hint_texts_omit_effectively_disabled_feature`**
   is the symmetric case: feature *enabled* in config, then
   forced-disabled on the live session. Asserts empty output.
   Old behavior would have returned the hint texts based on the
   stale config snapshot.

The `Arc::get_mut(&mut session).expect("session should not be shared")`
pattern at `:5446` and `:5460` is the right test escape hatch —
guards at runtime that no other Arc handles exist (so the mutation
is sound), and panics with a useful message if the session ever gets
cloned before the mutation.

## Risks / nits

1. **`session.features` mutation through `Arc::get_mut` is a
   test-only pattern that depends on no-Arc-cloning.** If a future
   refactor adds an `Arc::clone(&session)` somewhere in the
   `make_multi_agent_v2_usage_hint_test_session` helper or the
   surrounding test fixture, both new tests will start panicking at
   the `.expect("session should not be shared")` site. The panic
   message is good but the bug-shape is "test breaks for reasons
   unrelated to the SUT." Worth a comment near the helper warning
   future maintainers.

2. **No assertion that lock is *not* taken on the disabled path.**
   The structural improvement of lock-after-check is real but
   uncovered. A `parking_lot`-style `try_lock` assertion in a third
   test ("disabled feature returns without contention") would pin
   the perf-relevant invariant — otherwise a future "cleanup" PR
   that re-orders the check inside the lock would silently regress
   the call-site cost.

3. **The function lives on `Session` now but is still
   `pub(crate)` and called via `parent_thread.codex.session.<...>`
   chained traversal.** That's a tight coupling — every external
   call site needs to know the `codex.session` indirection. Worth
   exposing a thin pass-through on `CodexThread` (e.g.,
   `pub async fn multi_agent_v2_usage_hint_texts(&self) ->
   Vec<String> { self.codex.session.configured_...() }`) so the
   internal structure is hidden. Minor refactor.

4. **No coverage for the multi-write race.** If two parallel turns
   call this method while a feature override is being applied, the
   interleaving is undefined — could see hints from either pre- or
   post-override state. Probably fine in practice (feature
   overrides are not frequent), but a property-test or stress-test
   would clarify the policy.

## Verdict

**merge-as-is.** The fix is a model-class example of "the bug was the
wrong type owned the function" — by moving it to the type that owns
the live state, the stale-snapshot bug is *structurally impossible*
rather than fixed in-place. The two new tests are direct proofs of
both directions of the contract (enabled-in-effective-state-only and
disabled-in-effective-state-only). Lock-after-check is a real perf
improvement on the hot disabled path. Nits 1-4 are all post-merge
follow-ups, none blocking.

## What I learned

- "The function lives on the wrong type" is a recurring bug shape
  whenever a function reads from two pieces of state and one of them
  is on the type, the other is reached through indirection. The fix
  is always to move the function to the type that owns the
  most-frequently-changing piece of state — the indirection then
  goes the other way.
- `Arc::get_mut(&mut x).expect("x should not be shared")` is the
  right test pattern for "I need to mutate a normally-Arc'd value in
  a controlled test fixture." The runtime check guards against a
  future cloning regression; the panic message tells the next
  maintainer exactly which invariant they broke.
- `original_config_do_not_use` is a wonderful field name — it
  documents the policy (don't read this for runtime decisions) at
  the type level. When you find a call site that *does* read it for
  a runtime decision, that's a structural bug not a typo.
- Lock-after-check is a free perf win whenever the check can be
  done against lock-free state and the lock is otherwise cheap to
  skip on the early-return path. Worth pattern-matching at every
  `state.lock().await; if !condition { return ... }` site.
