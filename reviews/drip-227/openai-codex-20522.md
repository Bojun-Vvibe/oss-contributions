---
pr: openai/codex#20522
title: "Alias codex_hooks feature as hooks"
head_sha: 392117a4fbc331e1b22d0e9678cabadec08738da
verdict: merge-as-is
drip: drip-227
---

## Context

The `codex_hooks` feature flag was holding the canonical name of a
soon-to-be-stable subsystem. Maintainers want the user-facing key to be
the concise `hooks`, but existing `[features]` blocks in user
configs already write `codex_hooks = true|false` and must keep working
after the rename. This PR does the rename plus the legacy alias plumbing.

## Diff walkthrough

The single source-of-truth change is at
`features/src/lib.rs:782-786`: `Feature::CodexHooks` flips its
`key:` field from `"codex_hooks"` to `"hooks"`. The Rust enum variant
name `CodexHooks` is intentionally left alone — the wire-format key is
what users type, and Rust callsites referring to the variant don't need
to change.

The legacy alias is registered at `features/src/legacy.rs:49-54` by
appending one entry to the existing `ALIASES: &[Alias]` table. The
`legacy_feature_keys()` iterator and the existing `feature_for_key`
lookup table already handle alias resolution, so this is a one-line
declarative add — no new control flow.

The schema reflects the new key at
`core/config.schema.json:451` and `:3471`, both surfaced for the two
distinct shapes that include the feature map. Note both schema arms got
the new key but neither schema arm removes the legacy key — that's
correct because legacy aliases come in via the alias table at runtime
and don't need to be in the schema's recommended-input shape.

Tests at `features/src/tests.rs:270-274` pin the contract by exact
match: `feature_for_key("hooks")` and `feature_for_key("codex_hooks")`
both resolve to `Feature::CodexHooks`. The integration test fixtures
at `app-server/tests/suite/v2/hooks_list.rs:66,233,241` and
`core/tests/suite/hooks.rs:318` are migrated to the new canonical key,
exercising the new key end-to-end while the alias table covers the old
one.

## Risks

- No coverage of the `codex_hooks=true` arm in
  `app-server/tests/suite/v2/hooks_list.rs` (those fixtures all moved
  off the legacy key); the unit test in `tests.rs:272` is the only
  proof the legacy alias still resolves. Acceptable because alias
  lookup is a single hash-table lookup with no per-feature branching.
- Users with both `hooks` and `codex_hooks` set in the same config
  will hit whatever the existing alias-collision policy is —
  out-of-scope here.

## Verdict

**merge-as-is** — minimal, surgical, type-safe rename with the legacy
shape preserved through the existing alias mechanism. Test coverage
pins the bidirectional contract.
