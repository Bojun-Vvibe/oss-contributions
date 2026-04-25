---
pr: 18883
repo: openai/codex
sha: 7ede75de4ae4b5da9725f93006d577b139c00dcf
verdict: merge-as-is
date: 2026-04-26
---

# openai/codex#18883 — [codex] fix network context removal

- **URL**: https://github.com/openai/codex/pull/18883
- **Author**: tibo-openai
- **Files**:
  `codex-rs/core/src/context/environment_context.rs`,
  `codex-rs/core/src/context/environment_context_tests.rs`,
  `codex-rs/core/src/session/tests.rs` (+117/-2)

## Summary

When the user's network constraints are removed mid-session
(e.g. a cloud requirement layer drops its network policy block),
the diff between the previous and current `EnvironmentContext`
went from `Some(network)` to `None`. The serializer for the
None branch had a TODO that left it intentionally silent — so
the model never saw any signal that network access was now
disabled. It just stopped seeing the `<network>` block, which
is ambiguous (could mean "unchanged" or "removed"). The PR adds
an explicit `enabled: bool` field on `NetworkContext` and emits
`<network enabled="false" />` when network is removed, so the
model gets a clear "network is now off" signal.

## Reviewable points

- `codex-rs/core/src/context/environment_context.rs:8-11` — adds
  `enabled: bool` to `NetworkContext`. `pub(crate) fn new(...)`
  on line 16 always sets `enabled: true`, and a new private
  `disabled()` constructor on line 25-30 sets it false with
  empty allowed/denied lists. Constructor split is the right
  call: external callers can't accidentally construct a
  "disabled-but-with-allowed-domains" context.

- `environment_context.rs:91-95` — the diff transition logic.
  Old code:
  ```rust
  let network = if before_network != after.network {
      after.network.clone()
  } else { before_network };
  ```
  New code wraps the `after.network.clone()` with
  `.or_else(|| Some(NetworkContext::disabled()))`. Subtle but
  correct: this means whenever `after.network` is `None` *and*
  the network state changed (i.e. it was previously `Some`), we
  emit a synthetic `disabled` context instead of leaving the
  field as `None`. If both before and after are `None`, the
  `before_network != after.network` guard is false and the else
  branch passes through `None`, so we don't spuriously emit
  `<network enabled="false" />` on a turn where network was
  never relevant. Right both ways.

- `environment_context.rs:196-210` — the renderer now branches
  three ways:
  - `Some(network) if network.enabled` → emit full
    `<network enabled="true">` with allowed/denied children.
  - `Some(_)` (i.e. enabled=false) → emit
    `<network enabled="false" />`.
  - `None` → emit nothing (the original TODO behavior preserved
    for the never-set case).

  The match guard `if network.enabled` is the elegant bit — no
  need for an explicit if/else inside the Some arm.

- `environment_context_tests.rs:75-95` — direct unit test for
  the disabled rendering. Asserts the exact string with
  `<network enabled="false" />` and confirms no
  allowed/denied children appear. Tight.

- `session/tests.rs:4431-4504` — integration-level test
  (`build_settings_update_items_marks_network_disabled_when_network_is_removed`)
  rebuilds the previous context with a network requirement,
  rebuilds the current context without it, and asserts the
  emitted environment update item contains `<network
  enabled="false" />` *and* does not contain the previously
  allowed `api.example.com`. The "and not" assertion is
  important — it catches the bug where a transition might still
  echo old allowed-domain entries.

## Risks

- The `enabled` field is purely internal (`pub(crate)`) and
  default-derived, so any code path that constructs a
  `NetworkContext` via `Default::default()` will get
  `enabled: false` plus empty lists. Searched the diff for new
  Default uses — none introduced. But existing callers using
  `Default::default()` would now produce a "disabled" context
  instead of (effectively) nothing. Worth one quick `rg
  "NetworkContext::default"` in the rest of the codebase to
  confirm no such call exists; the diff doesn't show any.
- The renderer emits `<network enabled="false" />` only when
  the diff observed a transition. If the model context window
  later evicts the prior turn (where network was on), the model
  may not have any preserved memory of "network was once
  allowed and is now off" — only the current turn's
  `enabled="false"`. That's fine semantically (current state is
  what matters) but worth knowing for debugging.

## Verdict

`merge-as-is`. Clean, small, properly tested at both the unit
and integration level. The match-arm-with-guard pattern is the
idiomatic Rust way to handle the three-way render decision.

## What I learned

When a context block transitions from "set" to "unset",
*absence* in the serialized form is ambiguous — it could mean
"unchanged" or "removed". An explicit "now disabled" marker is
the right way to disambiguate, and it's worth having a sentinel
constructor (here `disabled()`) to prevent the obvious
construction bug of disabled-but-with-data.
