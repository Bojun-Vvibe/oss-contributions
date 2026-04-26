---
pr: 3611
repo: QwenLM/qwen-code
sha: 15fed468e9749746bdbe9127c84af8f4a6f3503a
verdict: merge-as-is
date: 2026-04-27
---

# QwenLM/qwen-code #3611 — fix(review): respect /language output setting for local reviews

- **Author**: yiliang114
- **Head SHA**: 15fed468e9749746bdbe9127c84af8f4a6f3503a
- **Size**: +2/-2 in `packages/core/src/skills/bundled/review/SKILL.md`.

## Scope

The `/review` skill prompt has a "Match the language of the PR" rule. For PR reviews this is correct — comments may be posted via `--comment`, so the PR's language is the right anchor. For **local reviews** (no PR target), there's no PR to match, so the rule defaults to whatever the model picks, which drifts. This PR amends the rule to fall back to the user's `/language output` preference (and then to input language) when there's no PR.

## Specific findings

- `SKILL.md:20` — first occurrence in the "Critical rules" header. The diff appends one sentence: "For **local reviews** (no PR), if the system prompt includes an output language preference, use that language; otherwise follow the user's input language." That's the right precedence: explicit-config > input-language > model default.
- `SKILL.md:531` — second occurrence at the bottom of the file in the closing reminders block, with the parallel sentence: "For **local reviews** (no PR), respect the user's output language preference if set; otherwise follow the user's input language." Slightly different wording from `:20` but semantically identical — fine.
- The two phrasings ("if the system prompt includes an output language preference" vs "respect the user's output language preference if set") could be unified to one canonical phrasing for less drift on future edits. Minor.
- PR-review path is explicitly preserved unchanged: "PR review behavior is unchanged" (PR body) — the new sentence only fires "for local reviews (no PR)".

## Risk

Negligible. Two-line text-only change to a skill prompt, additive in semantics, doesn't alter any code path or PR-review behaviour. Worst case: a model that interprets the appended sentence ambiguously falls back to the existing default, which is what users currently get.

## Verdict

**merge-as-is** — surgical doc-fix to a real and reported drift in local-review language behaviour, with the PR-review code path untouched. The two slightly-different phrasings at `:20` and `:531` would be nice to unify but it's nit-tier and not worth blocking on.
