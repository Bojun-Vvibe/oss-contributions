# openai/codex PR #19805 — split MultiAgentV2 usage hint into root vs subagent variants

- **PR**: https://github.com/openai/codex/pull/19805
- **Author**: @jif-oai
- **Head SHA**: `dfa038481dcc1351d8ab1806a88cd446438e27d5`
- **Size**: +289 / −4
- **Files**: `codex-rs/core/src/agent/control.rs`, `codex-rs/core/src/agent/control_tests.rs`, `codex-rs/core/src/config/mod.rs`, `codex-rs/core/src/config/config_tests.rs`, `codex-rs/core/src/session/mod.rs`, `codex-rs/core/config.schema.json` (and a new `session/multi_agents.rs` module)

## Summary

Adds two new optional `MultiAgentV2Config` fields — `root_agent_usage_hint_text` and `subagent_usage_hint_text` — alongside the existing single `usage_hint_text`. When forking a thread, the rollout-retain filter now drops the parent's previously-injected hint developer messages so the child receives a fresh hint matching its *own* config (root vs subagent), not the parent's stale text.

## Verdict: `merge-after-nits`

Right shape — the bug it fixes is real (a subagent forked from a root thread inherits the *root*'s injected hint text and gets a "you may delegate to subagents" line that doesn't apply). The strip-then-reinject pattern is correct. Two concerns: the strip uses string equality on full hint text, which is brittle, and the fallback when `parent_thread` is `None` reaches into `config.features.enabled(Feature::MultiAgentV2)` instead of trusting the call site.

## Specific references

- `codex-rs/core/src/agent/control.rs:401-432` — the new retain filter computes `multi_agent_v2_usage_hint_texts_to_filter` either by asking the parent thread for its `configured_multi_agent_v2_usage_hint_texts()` (good — authoritative) or, when no parent exists, by reading both hint fields from the *child's* own config. The latter branch is conservative for the "no parent" case but slightly wrong: if a user changed their config between the parent thread's start and the fork, the child's config is the right source for the *new* hint but the wrong source for the *old* hint that lives in `forked_rollout_items`. The parent-side `configured_multi_agent_v2_usage_hint_texts()` should be the authoritative path; the no-parent branch should match on a structural marker (developer role + a `multi_agent_v2_hint` annotation in the rollout item) rather than text equality.
- `codex-rs/core/src/agent/control.rs:418-431` — the `if let RolloutItem::ResponseItem(ResponseItem::Message { role, content, .. }) = item && role == "developer" && let [ContentItem::InputText { text }] = content.as_slice()` pattern is correct in shape. The single-element-slice match (`[ContentItem::InputText { text }] = content.as_slice()`) silently passes through any developer message that happens to have multiple content items, which today is fine but couples to an implementation detail of how hints are injected. Worth a comment explaining the invariant.
- `codex-rs/core/src/config/mod.rs:1600-1622` — `resolve_multi_agent_v2_config` extends the existing profile/base/default resolution chain to the two new fields with the same `profile.or_else(base).cloned().or(default)` shape. Mechanical; matches the pattern for `usage_hint_text`. Good.
- `codex-rs/core/src/agent/control_tests.rs:595-680` — the new test (`spawn_agent_can_fork_parent_thread_history_with_sanitized_items`) constructs a *parent* config with `"Parent root guidance."` / `"Parent subagent guidance."`, a *child* config with `"Child root guidance."` / `"Child subagent guidance."`, injects both parent hint messages into the rollout, then asserts the child fork strips both. The test correctly exercises the "parent-derived" branch via `configured_multi_agent_v2_usage_hint_texts().await`. There's no test for the no-parent branch (where the strip falls back to the child's own config); that's the branch flagged above as semantically dubious and a regression test would lock the chosen behavior.
- `codex-rs/core/src/config/config_tests.rs:7351-7430` — both fields appear in the base-only and profile-overrides-base test fixtures. The profile-override case correctly asserts that `Some("profile root hint")` wins when both base and profile set the field.

## Nits

1. Document the no-parent branch's semantics — is it "best-effort strip using child config" or "guaranteed strip using marker"? The diff doesn't say.
2. Either annotate hint-injection rollout items with a structural marker, or leave a comment at L418-422 explaining why text-equality is the chosen identity.
3. The single-element `[ContentItem::InputText { text }]` slice pattern silently lets any developer message with two content items through. Add a `// hints are injected as a single-text-item developer message; multi-item shapes are user content and must not be filtered` comment.

## What I learned

The "strip parent hint, reinject fresh child hint" pattern in fork paths is the right cure for hint drift across a fork boundary, but it makes the *identity* of an injected hint a load-bearing piece of state. Two design choices for that identity exist: by-content (text equality, what this PR uses) and by-marker (an annotation on the `RolloutItem` saying "this is an injected hint, drop on fork"). The marker version is more robust to config changes between fork events; the content version is simpler and survives serialization round-trips. This PR picks the simpler one and is correct today, but a follow-up to add a marker would harden it against the "user edits hint text mid-session" failure mode.
