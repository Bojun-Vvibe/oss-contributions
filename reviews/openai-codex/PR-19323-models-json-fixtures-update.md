# openai/codex PR #19323 — Update models.json and related fixtures

Link: https://github.com/openai/codex/pull/19323
State: MERGED · Author: (internal) · +173 / −41 · Files: multiple

## Summary
A fixture/data update to `models.json` and a small set of TUI snapshot fixtures.
Adds a new model entry (with full instructions templates and personality variants), tweaks
descriptions on `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.3-codex`, and `gpt-5.2`, and changes the
`default_reasoning_level` on `gpt-5.4` from `medium` to `xhigh`. TUI snapshot deltas are
mechanical (model display name `gpt-5.4` → `gpt-5.5`, an extra block of blank lines in the
guardian-review snapshot).

## Strengths
- Bundling the model-data change with its dependent snapshot updates in a single PR keeps CI
  green and avoids the "snapshot churn in a sibling PR" anti-pattern.
- The new model's `available_in_plans` array is exhaustively enumerated rather than relying on
  a wildcard, which makes future plan rollouts auditable.
- Description normalization (trailing-period consistency on `gpt-5.2`) is the kind of small
  hygiene fix worth doing in a data-only PR.

## Concerns / Risks
- **Default reasoning level promotion is a silent behavior shift.** Changing
  `gpt-5.4.default_reasoning_level` from `medium` to `xhigh` is a runtime contract change for
  every caller who omits the explicit reasoning-level field. Latency, cost per call, and
  token usage will all rise materially. Nothing in the diff (or, judging by the snapshot
  changes, anywhere else) flags this as a behavior change for end users. A release note or a
  separate PR with a clear changelog entry would have been the right shape; rolling it into
  a "models.json update" PR risks future blame-archaeology when someone tracks down a bill
  spike to this commit.
- **Description copy regressions.** `gpt-5.4` description goes from "Latest frontier agentic
  coding model" to "Strong model for everyday coding"; `gpt-5.3-codex` from "Frontier
  Codex-optimized agentic coding model" to "Coding-optimized model". These are net
  *demotions* in marketing language, which suggests `gpt-5.5` is supersedng them. If so, the
  PR should also set the new model as the default in any selector logic — otherwise users on
  defaults will keep getting the now-deprecated descriptor without seeing the upgrade path.
- **Snapshot blank-line drift.** The guardian-review snapshot acquires several leading blank
  lines. That is almost certainly the result of an unrelated TUI layout change auto-applied
  during regeneration. Mixing layout drift into a "data update" PR makes future bisection
  harder; ideally the snapshot would have been re-run with no layout changes pending.
- **`personality_default` defined as empty string** while `personality_friendly` and
  `personality_pragmatic` carry real text means the consumer of `instructions_variables`
  must handle an empty value as a special case, not as "skip personality block". If the
  template substitutes raw, the rendered prompt may have a stray blank line where the
  personality block was. Worth a snapshot test.
- **Plan list duplication risk.** `enterprise` and `enterprise_cbp_usage_based` and
  `self_serve_business_usage_based` appear together; if any of these are aliases or one
  implies the other, the entitlement code may double-grant. Fixture changes don't surface
  this, but the list grew without an obvious source-of-truth comment.

## Suggested follow-ups
1. Document the `default_reasoning_level` change in release notes; consider gating the
   shift behind a config flag for one release.
2. Audit whether `gpt-5.5` should be the new default model in selectors and TUI hint text.
3. Add a snapshot test that renders the `instructions_template` with an empty
   `personality_default` to lock the no-personality path.
