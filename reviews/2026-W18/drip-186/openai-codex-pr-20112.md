---
pr: openai/codex#20112
sha: ab4f1d38ba6103e57ca43d587b47d3daac470f3f
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# Soften skill description budget warnings

URL: https://github.com/openai/codex/pull/20112
Files: `codex-rs/core-skills/src/render.rs`,
`codex-rs/core/src/session/tests.rs`,
`codex-rs/app-server/tests/suite/v2/turn_start.rs`,
`codex-rs/tui/src/chatwidget/tests/app_server.rs`
Diff: 53+/29-

## Context

The skill-budget warning surface was both (a) too noisy (firing at the
10-character truncation threshold) and (b) too alarmist (every message
prefixed with "Warning:" and a numeric "truncated by an average of N
characters per skill" tail that read as a hard error in chat UIs).
Three changes in this PR:

1. `SKILL_DESCRIPTION_TRUNCATION_WARNING_THRESHOLD_CHARS` raised from
   10→100 at `render.rs:19` so cosmetic single-word truncations no
   longer trip the warning.
2. The truncated-but-included copy at `render.rs:21-22` flips from a
   numeric tail to two static, calmer strings
   (`SKILL_DESCRIPTION_TRUNCATED_WARNING` and the percent-aware
   `_WITH_PERCENT` variant), with the budget arm switching at
   `:234-241` based on `SkillMetadataBudget::Tokens` vs `Characters`.
3. `SKILL_DESCRIPTIONS_REMOVED_WARNING_PREFIX` at `:23-24` and the
   downstream `WarningEvent.message` strings drop the `"Warning: "`
   prefix entirely (the surrounding UI surface already renders the
   warning level).

The averaging arithmetic at `:436-444` also fixes a real bug: the
prior denominator `truncated_description_count` made the average
spike to absurd values when only a few skills were truncated; the new
`total_count` denominator gives a meaningful "how much room are we
short, on average across all skills" number, and the early-return
guard now also short-circuits on `truncated_description_chars == 0`.

## What's good

- Threshold bump 10→100 at `render.rs:19` is the right magnitude for
  "user-noticeable truncation" rather than "compiler-noticeable
  truncation".
- The static-string approach at `:21-22` is correct for a UX message
  that doesn't actually depend on runtime values for actionable advice
  ("Disable unused skills or plugins to leave more room for the rest"
  is the action regardless of N).
- The averaging-denominator fix at `:436-444` is a load-bearing
  correctness change wearing a UX-polish disguise — the prior formula
  divided by truncated count, so a single 200-char truncation across
  100 skills reported as "200 chars per skill" instead of "2 chars per
  skill", making the old warning literally lie about its magnitude.
- Test-side updates symmetric and complete: the new
  `budgeted_rendering_token_budget_truncation_warning_mentions_two_percent`
  test at the bottom of `render.rs` locks the percent-bearing variant,
  the existing test at `:1051-1098` gets a focused rewrite using a
  `long_description = "a".repeat(250)` + empty-skill pair to exercise
  the `total_count=2, truncated_count=1` path that hits the
  denominator fix.
- Three downstream test sites
  (`session/tests.rs:5610,5721,5732`, `turn_start.rs:336`,
  `app_server.rs:194,204`) updated in lockstep to the
  prefix-stripped strings — no orphaned `"Warning: "` assertions.

## Nits / follow-ups

- The two static strings `SKILL_DESCRIPTION_TRUNCATED_WARNING` vs
  `..._WITH_PERCENT` differ only in the literal `"2%"` token — the
  `2%` is also encoded in `SKILL_METADATA_CONTEXT_WINDOW_PERCENT` at
  `:18`. If that constant is ever bumped to 3% or 5%, the warning
  string will lie. Either build the percent suffix at runtime via
  `format!()` or add a `const_assert!(SKILL_METADATA_CONTEXT_WINDOW_PERCENT == 2)`
  near the string definitions to make the coupling explicit.
- The two test-name pairs `truncated_description_chars == 28` →
  `== 202` at `:1075-1077` are correct against the new fixture but
  the magic number `49` in `Characters(minimum_cost + 49)` at
  `:1062` is silently encoding "the long-description budget that
  produces exactly the 202-char truncation reported on next line" —
  a one-line comment naming the relationship would save the next
  reader 5 minutes.
- The early-return guard at `:436` now reads `total_count == 0
  || truncated_description_chars == 0` — the second clause is a
  defensive check that should never fire given the call site at
  `:230-241` already gates on `report.average_truncated_description_chars()
  > THRESHOLD_CHARS`. Worth either commenting why the belt-and-suspenders
  is intentional or removing it for clarity.
- No CHANGELOG entry for what is effectively a user-visible UX wording
  change in the most-noisy diagnostic surface — operators with log
  scrapers or alert-on-`"Warning:"` rules will silently lose
  visibility into both warning classes.

## Verdict

`merge-after-nits` — UX softening is well-scoped and the quietly
included averaging-denominator correctness fix is the right kind of
"happens to also be a bug fix" piggyback, but the percent-token
coupling between the constant and the literal string deserves either
a runtime format or a static_assert before the next budget tweak
makes the warning lie about itself.
