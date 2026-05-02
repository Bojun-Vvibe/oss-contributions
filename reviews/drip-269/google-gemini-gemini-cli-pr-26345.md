# google-gemini/gemini-cli #26345 — feat: add secure non-interactive multi-agent orchestration

- URL: https://github.com/google-gemini/gemini-cli/pull/26345
- Head SHA: `e62a8d9187bfa279765aac652cb5f0e14ac96cf8`
- Author: @rushikeshsakharleofficial
- Stats: +604 / -7 across 5 files

## Summary

Adds a sequential, role-based (planner → researcher → coder → tester →
reviewer) non-interactive orchestration path that reuses the existing
`runNonInteractive` execution surface. Caps agent count, supports
`dryRun`, and explicitly does NOT expose CLI flags (the
`argv.multiAgent` branch in `gemini.tsx` is currently unreachable from
user input). Ships a 142-line `interactive-multi-agent-plan.md` design
document describing the path toward a real interactive TUI experience
in a future PR.

## Specific feedback

- `packages/cli/src/gemini.tsx:797-815` — adds the `if (argv.multiAgent)`
  branch around the existing `runNonInteractive(...)` call. Cleanly
  isolated; the non-multi-agent path is preserved byte-for-byte. But
  per the PR body itself: "CLI flags are not exposed yet to keep this
  PR minimal" — meaning `argv.multiAgent` / `argv.multiAgentRoles` /
  `argv.multiAgentMax` / `argv.multiAgentDryRun` are read here but
  never set anywhere user-reachable. That's dead code from the user's
  perspective, which is unusual to land in main.
- `docs/cli/interactive-multi-agent-plan.md:1-142` — the design doc is
  thorough and the right shape: clear non-goals (no uncontrolled
  parallel writes, no permission bypass), phased plan, security
  considerations, and a recommended next-PR shape. The doc itself is
  arguably the most valuable artifact in this PR.
- Specifically the doc's own recommendation in its closing section is:
  "The current PR should either: (1) Become a small design/scaffold PR
  that does not expose unreachable CLI flags; or (2) Be updated to
  include minimal CLI flag typing plus tests, clearly labeling it as a
  non-interactive precursor." That's exactly the question reviewers
  need to answer here, and the author is flagging it themselves.
- The `multiAgent/nonInteractiveMultiAgent.ts` module (parseMultiAgent
  Roles, normalizeMaxAgents, runNonInteractiveMultiAgent) is imported
  but the actual file contents weren't visible in the truncated diff —
  reviewers should specifically check: (a) sandbox/policy reuse claim,
  (b) cancellation behavior, (c) prompt-construction safety in dryRun.
- Security framing in the PR body is correct in spirit ("preserve all
  existing security controls", "no new permissions") but prompt-text
  alone is not an enforcement boundary — the design doc says exactly
  this at line 113. The actual enforcement must come from the
  scheduler / policy engine, not from instructions like "you are a
  planner, do not write files".

## Verdict

`needs-discussion` — the author has explicitly flagged the
"unreachable-CLI-flags vs. minimal-flag-typing" decision in the design
doc. Maintainers should pick path (1) or (2) before merge. The design
doc is solid and worth landing on its own; the scaffold code without
reachable CLI flags is the open question.
