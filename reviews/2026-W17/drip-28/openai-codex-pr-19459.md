# PR #19459 — Enable unavailable dummy tools by default

- Repo: openai/codex
- Head SHA: 44fca11600890264b5551cd79915dbaa0cb5be86
- Files touched: 2 (+5 / -8)

## Summary

Two-line feature-flag flip in the codex-features registry:
`Feature::UnavailableDummyTools` graduates from
`Stage::UnderDevelopment` / `default_enabled: false` to
`Stage::Stable` / `default_enabled: true`. The matching test in
`codex-rs/features/src/tests.rs:160-166` is renamed and inverted
to assert the new state.

## Specific findings

- `codex-rs/features/src/lib.rs:845-850` — the registry entry
  changes:
  ```rust
  FeatureSpec {
      id: Feature::UnavailableDummyTools,
      key: "unavailable_dummy_tools",
-     stage: Stage::UnderDevelopment,
-     default_enabled: false,
+     stage: Stage::Stable,
+     default_enabled: true,
  }
  ```
  Test name changes from
  `unavailable_dummy_tools_is_under_development_and_disabled_by_default`
  to `..._is_stable_and_enabled_by_default` and the body is
  inverted to match.
- This is purely an operator-default flip; the feature itself is
  not being defined or extended in this PR. The runtime contract
  for `unavailable_dummy_tools` is owned elsewhere (consumers
  read the flag from the features registry).

## Risks / nits

- The PR body says only "Mark as stable / enable by default." It
  doesn't reference any rollout doc, prior beta-flag bake-time,
  or known-issue tracker. For a Stable + on-by-default flip,
  reviewer should confirm:
  1. The feature has been on for the cohort long enough that
     "enable by default for everyone" is justified.
  2. Any operator that *opted into* `unavailable_dummy_tools=
     false` because of a known issue still has a working escape
     hatch (the `default_enabled` flip doesn't remove the
     ability to set it `false` explicitly via config — but
     worth confirming there's no `Stage::Stable` invariant that
     forbids opt-out elsewhere).
- The diff is so small that it's effectively a paper trail for
  a decision made elsewhere. The risk surface is entirely in
  the consumer code that reads the flag — none of which
  appears in this diff.
- No release-note line in the PR body. A user-default change
  for a tool-availability feature deserves at least one bullet
  in the changelog so anyone debugging "where did this dummy
  tool come from" can grep their way back to the change.

## Verdict

**merge-after-nits** — The flag mechanics are correct
(stage + default flip + matching test). Block on a release-
note line and an explicit confirmation in the PR body that
the bake period was sufficient and that opt-out via config
still works.

## What I learned

The smallest possible PR shape — a four-line registry edit
plus a renamed test — is also where regressions are most
likely to be invisible. The diff carries zero context about
the consumer behavior change being unlocked, so the reviewer
has to either know the feature already or grep `lib.rs` for
the flag's call sites. A two-line "what unavailable_dummy_
tools does when on" sentence in the PR body would close that
gap with no engineering cost.
