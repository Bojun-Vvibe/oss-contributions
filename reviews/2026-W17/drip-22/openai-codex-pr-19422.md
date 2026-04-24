# PR #19422 — Clarify bundled OpenAI Docs upgrade guide wording

- URL: https://github.com/openai/codex/pull/19422
- Author: kkahadze-oai
- Head SHA: `212bd457fef5205d2f5feca4b8388c094dcab89c`

## Summary

Editorial-only change to the bundled `openai-docs` skill's `upgrade-guide.md`.
Replaces directive language ("Always emit", "preserve it first") with
recommendation language ("Include", "recommend preserving"), and swaps internal
snake_case prompt-block tokens (`research_mode`, `tool_persistence_rules`,
`completeness_contract`, etc.) for natural-language descriptions of the
underlying behaviors. No code, schema, or skill-loader changes.

## Specific callouts

- `codex-rs/skills/src/assets/samples/openai-docs/references/upgrade-guide.md:60-65`
  — Output rule softening. The old wording was a hard contract ("Always emit
  a starting `reasoning_effort_recommendation` for each usage site"); the new
  wording ("Include …", "recommend preserving …", "recommend not adding …")
  delegates the decision to the operating model. This is consistent with the
  rest of the guide treating the model as the policy enforcer rather than the
  skill, but it does mean any downstream tooling that grepped for the literal
  string `Always emit` to gate behavior will need updating. Worth a one-line
  changelog note.
- `upgrade-guide.md:99-105` — The four workflow defaults
  (research / dependency-aware / coding / multi-agent) were previously expressed
  as a closed set of named blocks (`citation_rules`, `empty_result_handling`,
  `tool_persistence_rules`, `dependency_checks`, `verification_loop`,
  `terminal_tool_hygiene`, `completeness_contract`, `parallel_tool_calling`).
  The new language describes the *outcomes* the model should produce instead
  of naming reusable building blocks. Two consequences:
  1. Skills/agents that built a prompt-block library keyed on those names can
     no longer cross-reference this guide as the source of truth — that
     coupling is now severed by design, but please confirm there's no
     in-tree consumer (`rg "research_mode|tool_persistence_rules|completeness_contract"`
     across `codex-rs/`).
  2. The "add `parallel_tool_calling` only when retrieval steps are truly
     independent" caveat is gone in the new text. That nuance was load-bearing
     for tool-heavy agents — losing it risks more aggressive parallelism. If
     the prompting guide already covers this, fine; if not, port the caveat
     into the new "explicit tool budgets, stop conditions" line.
- The `git diff --check` test plan only catches whitespace; no test verifies
  the bundled-skill content actually round-trips through whatever loader
  consumes it. Acceptable for prose-only edits.

## Risks

- Any out-of-tree skill registry pinned to the old token names will silently
  diverge from the bundled guidance. Low blast radius (this is a bundled
  internal skill copy, not a public API), but worth flagging in the PR
  description for downstream consumers tracking the upstream OpenAI Docs
  skill.
- Loss of the `parallel_tool_calling` independence caveat is the only
  semantic regression in an otherwise editorial change.

## Verdict

**Verdict:** merge-after-nits

Restore the "only when retrieval steps are truly independent" qualifier
(or its equivalent) in the dependency-aware bullet — that's the one place
the new wording loses safety guidance the old wording carried. Everything
else is a clean tightening.
