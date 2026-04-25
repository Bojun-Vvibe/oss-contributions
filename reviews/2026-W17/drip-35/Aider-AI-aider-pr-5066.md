# Aider-AI/aider #5066 — Polyglot benchmark: Claude Opus 4.7 new #1 at 93.3%

- **Repo**: Aider-AI/aider
- **PR**: [#5066](https://github.com/Aider-AI/aider/pull/5066)
- **Head SHA**: `a5b0f6db93c8fb1ebbd325363dbc52f64a60c9da`
- **Author**: anishesg
- **State**: OPEN (+28 / -0)
- **Verdict**: `merge-after-nits`

## Context

Pure data-only PR: appends a single new run entry to
`aider/website/_data/polyglot_leaderboard.yml`. No code, no logic
changes, just a new YAML object claiming a new top score on the
polyglot leaderboard. Reviewing this is reviewing the *data integrity*
and *reproducibility* of the claim, not implementation.

## Design (such as it is)

The diff at
`aider/website/_data/polyglot_leaderboard.yml:1857-1884` is a single
list item with the standard schema fields. Spot-checked the field set
against the prior entry above it (lines 1845-1854): every field
present in the existing schema is supplied here too, with consistent
naming (`pass_rate_1`, `pass_rate_2`, `pass_num_1/2`,
`percent_cases_well_formed`, `error_outputs`, `num_malformed_responses`,
`num_with_malformed_responses`, `user_asks`, `lazy_comments`,
`syntax_errors`, `indentation_errors`, `exhausted_context_windows`,
`prompt_tokens`, `completion_tokens`, `test_timeouts`, `total_tests`,
`command`, `date`, `versions`, `seconds_per_case`, `total_cost`).
Nothing extra, nothing missing. YAML indentation matches.

Internal arithmetic checks:

- `pass_num_2: 210` / `total_tests: 225` = 93.33% — matches `pass_rate_2: 93.3`. ✓
- `pass_num_1: 146` / `225` = 64.88% — matches `pass_rate_1: 64.9`. ✓
- `total_cost: 26.2738` USD across `prompt_tokens: 3357354` +
  `completion_tokens: 528139` ≈ 3.89M tokens. At Bedrock Opus pricing
  (~$15/M input, ~$75/M output) that's ~$50 input + ~$40 output =
  ~$90, *not* $26. Either the user's `total_cost` is wrong, or the
  AWS Bedrock global inference profile is at a heavily discounted
  rate, or there's a credit applied. Worth the maintainer asking the
  author to clarify before merging — the leaderboard currently uses
  `total_cost` for cost-per-case rankings and a 3-4x understatement
  would be misleading. **This is the main thing I'd flag.**
- `seconds_per_case: 18.8` × 225 = 4,230s = 70.5 minutes total wall
  clock. Plausible for adaptive thinking on Opus.

## Risks / Nits

1. **`total_cost` looks low.** See above. Either confirm it's real
   (Bedrock provisioned-throughput discount, AWS credits, or the
   global inference profile being cheaper than I'm assuming) or
   recompute. If credits were applied, the cost should arguably be
   the list price, not the post-credit number, since the leaderboard
   is meant to inform other users' purchasing decisions.

2. **`commit_hash: 0189cf4`** — short SHA only. If the leaderboard
   tooling resolves these against the benchmark repo, fine; otherwise
   prefer the full 40-char SHA for unambiguity.

3. **`dirname: 2026-04-24-09-29-50--full-opus47-adaptive`** — matches
   the prevailing convention (`YYYY-MM-DD-HH-MM-SS--<slug>`). No
   reproducibility artifact (logs, scoring CSV) is referenced in the
   PR. Other entries on the leaderboard typically link to a results
   tarball; consider asking the author to upload one for
   verifiability — especially since this is now-rank-1.

4. **Adaptive thinking parametrization is opaque.** The model field
   says "(adaptive thinking)" but the `command` field is just
   `aider --model bedrock/global.anthropic.claude-opus-4-7` with no
   thinking-budget flag. If the adaptive mode is on by default for
   this model in aider's wiring, document that in a comment; if it
   required additional flags, those should appear in `command`.

5. **Self-reported "all-time #1"** is fine; the leaderboard sorts by
   `pass_rate_2` and the renderer will adjudicate.

6. **No syntax errors / 100% well-formed** is unusual but plausible
   for a top-tier model with diff edit-format on a clean run. Not
   suspicious, but worth a sanity-check that `error_outputs: 0` and
   `num_malformed_responses: 0` are computed by the same scoring
   pipeline as other entries, not hand-counted.

## Verdict rationale

`merge-after-nits`. The data shape is correct and the arithmetic
checks out for everything except `total_cost`, which I'd want the
author to either confirm or revise before this lands as the new #1
entry. The other items (full SHA, results tarball, adaptive-thinking
docs) are nice-to-have follow-ups that don't block.

## What I learned

For a leaderboard PR, the only meaningful review surface is
arithmetic consistency and reproducibility of the claim. The
`total_cost` field is interesting because it conflates list price,
discounted price, and any credits — leaderboards that compare
cost-efficiency across providers should standardize this, otherwise
"$26 for 225 cases" can mean wildly different things on Bedrock vs.
direct Anthropic vs. OpenRouter.
