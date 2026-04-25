# openai/codex PR #19514 — Fix codex-rs README grammar

- **URL:** https://github.com/openai/codex/pull/19514
- **Author:** @etraut-openai
- **State:** MERGED 2026-04-25 (1m after open — fast-track)
- **Base:** `main`
- **Head SHA:** `b316fdd`
- **Size:** +1 / -1
- **File:** `codex-rs/README.md`

## Summary of change

Single-word grammar fix in the workspace README. The "core/" crate
description previously read:

> Ultimately, we hope **this to be** a library crate that is generally
> useful…

Replaced with:

> Ultimately, we hope **this becomes** a library crate that is generally
> useful…

That's the entire diff.

## Findings against the diff

- **codex-rs/README.md L97**: the original phrasing
  "we hope this to be" is grammatically wrong (no main verb after
  "hope" — needs `[infinitive]` or `[that-clause]`). The new phrasing
  "we hope this becomes" is a `that`-clause with elided `that`, which
  is correct and idiomatic.
- Alternative phrasings considered (and equivalent in correctness):
  - "we hope to make this a library crate"
  - "we hope this will become a library crate"
  - "our hope is that this will be a library crate"

  The author picked the shortest valid form, which is the right call
  for a list-item bullet.
- One-minute open-to-merge with no review attached. For a one-word
  README diff that is by-construction risk-free, this is the right
  amount of process. The `apply-patch` round-trip and CI cycle are
  the only "review" needed.

## Verdict

**merge-as-is** (already merged)

Nothing to add. This is a model "small fix lands fast" PR.

## What I learned

A README grammar fix is the lower-bound calibration point for review
process: if your CI takes longer than your review for a one-word
fix, your review process is overweighted relative to your test
process; if your review takes longer than your CI for a one-word fix,
your review process is underweighted relative to your read-the-diff
process. One minute open-to-merge against an OpenAI-internal repo is
a healthy signal that the team's confidence in `git revert` is high
enough that they're not paying review tax on changes whose blast
radius is provably zero.
