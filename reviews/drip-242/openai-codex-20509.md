# Review: openai/codex #20509 — [codex] Remove workspace owner usage nudge gate

- URL: https://github.com/openai/codex/pull/20509
- Head SHA: `7ef6f7eaadd82601cc64d3b3ad9b65163134f5a4`
- Files: 3 (`features/src/lib.rs`, `tui/src/chatwidget.rs`, `tui/src/chatwidget/tests/status_and_layout.rs`)
- Size: +4/-81

## Summary of intended change

The `WorkspaceOwnerUsageNudge` feature flag gated three TUI behaviors around
"workspace member hit a usage limit, prompt to ask owner for credits" UX.
The flag has fully rolled out, so this PR:

1. Flips the flag's `Stage` from `UnderDevelopment` to `Removed` in
   `features/src/lib.rs:1071` while keeping the `Feature::WorkspaceOwnerUsageNudge`
   variant present (with updated doc comment at `:224-225`) so old configs
   and Statsig overrides containing the key still parse without error.
2. Deletes the helper `workspace_owner_usage_nudge_enabled()` and inlines
   "always true" at the three call sites in
   `chatwidget.rs:on_rate_limit_error` (`:2988-2998`), `open_workspace_owner_nudge_prompt`
   (`:7224-7232`), `set_workspace_owner_nudge_email_in_flight` (`:7282-7290`),
   and `complete_workspace_owner_nudge_email` (`:7298-7306`).
3. Deletes
   `workspace_owner_usage_nudge_flag_disabled_keeps_generic_rate_limit_error`
   from `chatwidget/tests/status_and_layout.rs:511-567` since the
   flag-disabled path is no longer reachable.
4. PR body links a companion change
   (`openai/openai#876351`) to remove the Statsig rollout gate after this
   change is in production.

## Review

### The "Removed" stage as a no-op compatibility surface

This is the right discipline. The variant `Feature::WorkspaceOwnerUsageNudge`
is still in the enum, the `FeatureSpec` is still in `FEATURES`, but
`stage: Stage::Removed` plus `default_enabled: false` means: anyone who has
the flag explicitly enabled in `~/.codex/config.toml` or via an override
won't get a parse error, but the flag has zero behavioral effect. That keeps
the rollout reversible (revert this PR and the gate comes back) and avoids
the "three-week deprecation cycle" anti-pattern. Worth confirming at
`features/src/lib.rs:1067-1073` that `Stage::Removed` is in fact a real,
distinct stage value (vs being introduced here). A quick grep for
`Stage::Removed` elsewhere in `features/src/lib.rs` will confirm prior
precedent for the pattern.

### Deletions in `chatwidget.rs`

All four deletions are correct and structurally identical: each removed an
`if !self.workspace_owner_usage_nudge_enabled() { ... return; }` early-exit
guard that took the "fall back to generic behavior" branch. With the gate
gone, the function bodies simply do the workspace-owner-aware thing
unconditionally, which is the desired post-rollout behavior. The deleted
helper at `chatwidget.rs:2988-2992` had no other callers (verifiable via the
PR's own diff — only the four sites above touched it).

One subtle correctness point: at
`chatwidget.rs:7298-7306`
(`complete_workspace_owner_nudge_email`), the deleted gate ran *after*
`self.add_credits_nudge_email_in_flight.take()`. So even pre-PR, if the
gate was disabled, the in-flight state was still drained — i.e. removing
the gate doesn't change the side-effect ordering on that path. Good.

### Test deletion

The deleted test
`workspace_owner_usage_nudge_flag_disabled_keeps_generic_rate_limit_error` at
`status_and_layout.rs:514-563` was specifically asserting the negative
behavior under the disabled flag (that `rendered.contains("Usage limit
reached.")` for the generic path). With the flag forced on, that assertion
no longer applies and keeping the test would mean keeping the (now
unreachable) generic branch. Correct to delete.

What's *not* in this PR but should be: a positive snapshot test pinning the
post-rollout behavior, i.e. that the workspace-owner prompt is now always
rendered for `RateLimitErrorKind::UsageLimit` +
`WorkspaceMemberUsageLimitReached`. The test
`rate_limit_switch_prompt_popup_snapshot` at `:511` (just above the
deleted block) probably already covers some of this surface, but worth
confirming it specifically asserts the workspace-member path.

## Verdict

**merge-after-nits**

Wants:
- Confirm in PR body that a positive snapshot test exists for the
  workspace-member-usage-limit path now that the negative test is gone
  (or add one); the diff currently only deletes coverage without adding
  any.
- Confirm that `Stage::Removed` is a pre-existing variant with prior
  precedent; if this PR introduces it, brief one-line doc comment on the
  enum variant explaining the "kept for config-parse compatibility, no
  behavior" intent.
- Optional: a follow-up tracking issue for full removal of the
  `Feature::WorkspaceOwnerUsageNudge` variant once the companion Statsig
  cleanup at `openai/openai#876351` has been deployed long enough to retire
  the compatibility surface entirely.

The structural shape of the cleanup is exactly right.
