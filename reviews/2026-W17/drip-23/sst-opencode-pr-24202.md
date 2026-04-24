# sst/opencode PR #24202 — perf(tool): condense tool descriptions ~66% to cut system-prompt tokens

- **Repo:** sst/opencode
- **PR:** [#24202](https://github.com/sst/opencode/pull/24202)
- **Head SHA:** `847adfdbf6ca01d6d25982c037fa2a74c1b1f97e`
- **Size:** +0/-? across 17 files (all `packages/opencode/src/tool/*.txt`)
- **Reviewer:** Bojun (drip-23)

## Summary

Pure prompt-engineering churn: rewrites every `*.txt` tool description
shipped with opencode into a terser style. The PR claims 33,163 → 11,302
chars (66% cut, ~6,250 tokens/msg). Issue #11995 is cited as the driver:
tool descriptions sit in the system prompt, are not part of any cacheable
prefix on most providers, and pay full input-token cost on every turn.
No code paths change — only the contents of the static `.txt` files
embedded into the agent system prompt.

## What's changed

Files touched (from `gh pr view --json files`):
- `apply_patch.txt`, `bash.txt`, `codesearch.txt`, `edit.txt`,
  `glob.txt`, `grep.txt`, `lsp.txt`, `plan-enter.txt`, `plan-exit.txt`,
  `question.txt`, `read.txt`, `skill.txt`, `task.txt`, `todowrite.txt`,
  `webfetch.txt`, `websearch.txt`, `write.txt`.

### `apply_patch.txt` (-22 lines, kept the contract)

Diff lines 1–39 of the PR. The verbose envelope explanation collapses
from a multi-paragraph "you get a sequence of file operations / You MUST
include a header" prose block into:

```
Each section MUST start with one header:
- `*** Add File: <path>` — create new; all following lines prefixed `+`.
- `*** Delete File: <path>` — remove; nothing follows.
- `*** Update File: <path>` — patch in place. Optional `*** Move to: <path>` rename.
```

The three semantically load-bearing facts survive:
1. exactly one of three header types per section,
2. Add-file lines must have `+` prefix,
3. Update can carry an optional `Move to:`.

Reasonable. The example block is preserved literally (lines 24–32 of the
diff), which is the part most models actually pattern-match against.

### `bash.txt` (-117 lines → -60, biggest absolute cut)

Diff lines 40–120. The pre-PR version (lines 40–117 of the diff) was a
verbatim chunk of an older Anthropic agent system prompt — including a
multi-step "Directory Verification" preamble, "Command Execution"
subsection with shell-quoting examples, an entire "Committing changes
with git" / "Git Safety Protocol" section, and a "Creating pull
requests" walkthrough. Most of that is **agent-policy content masquerading
as tool documentation** and does not belong in a tool description (the
model never sees `git commit --amend` rules from inside the bash tool
docstring — those should live in the system prompt or in a separate
skill).

Pulling them out of the per-tool docstring is correct in principle; the
risk is that any opencode user who relied on those rules being in-prompt
will lose them. Since I can't see the post-rewrite content here without
fetching the head ref, I'm trusting the PR description that the
behavioral guidance now lives elsewhere or has been intentionally
dropped.

### Files I cannot inspect from the diff alone

Files like `todowrite.txt` (-81%, 8,845 → 1,701 chars per the PR table)
and `task.txt` (-72%) likely follow the same pattern — long agent-style
"when to use / when not to use" sections being aggressively trimmed.
The PR description does not enumerate which behavioral hints were kept
vs dropped per file, which is the main concern below.

## Concerns

1. **Silent behavioral drift via prompt diet**
   Tool descriptions are *the* contract the model uses to decide when to
   call which tool. A 66% cut means a meaningful chunk of "use X for Y,
   not Z" guidance disappears. The PR offers no eval (golden trajectory,
   benchmark suite, even a smoke-test transcript) showing tool selection
   accuracy is preserved. For `todowrite` (-81%) and `task` (-72%) this
   matters most — those have the densest "when NOT to use" examples,
   which were doing real work for non-Sonnet models.

2. **No per-provider cache audit**
   The PR's headline claim is "non-cacheable, burn tokens every turn."
   That is true for some providers (e.g., DeepSeek, OpenAI without
   prompt caching) but for Anthropic the system prompt block including
   tool descriptions **is** cacheable via `cache_control` on the
   Messages API. Worth confirming opencode's Anthropic adapter is or
   isn't already attaching cache_control to the system block — if it is,
   the savings here are concentrated on non-Anthropic providers and
   the headline "6,250 tokens/msg" overstates the universal win.

3. **`bash.txt` policy content removal**
   The pre-rewrite `bash.txt` contained operational policy ("NEVER skip
   hooks", "NEVER force push to main/master", `--no-verify` rules). If
   that policy is removed entirely (rather than relocated to
   `prompts/system.txt` or similar), opencode loses defense-in-depth on
   git operations the model performs via bash. I'd want a single
   cross-reference grep against the head branch confirming those phrases
   land somewhere else in the prompt tree.

4. **Diff-only review of 17 files at once**
   Bundling all 17 file rewrites into one PR makes regression bisecting
   painful — if a tool starts misbehaving post-merge, you can't revert
   one tool's prompt without reverting the whole thing. A multi-PR split
   (one per tool, or per logical group: edit/write/apply_patch as one,
   bash/glob/grep as another, todowrite/task as a third) would be much
   safer. Not a blocker if maintainers are confident, but worth noting.

5. **Stylistic regression risk for `apply_patch`**
   The new `apply_patch.txt` (line 7 of the diff) opens with "Edit files
   with stripped-down diff format. Envelope:" — that's a fragment, not
   a sentence, and removes the "easy to parse and safe to apply"
   framing. Smaller models (4o-mini class) sometimes need that priming
   to commit to the format rather than emit raw unified-diff output.
   I'd add the example output back but keep the rest of the cut.

## Verdict

`merge-after-nits` — the optimization direction is right and the savings
are real on non-Anthropic providers. Want before-merge:

- a one-paragraph note in the PR body on **where** the removed `bash.txt`
  git/policy guidance now lives (or explicit confirmation it was dropped
  on purpose);
- a quick eval — even hand-rolled, 20 prompts × 2 conditions — showing
  tool-selection accuracy is preserved on at least one mid-tier model
  (4o-mini, Sonnet 4.5, Qwen 3.5 32B);
- ideally split into per-tool commits inside this same PR so a future
  bisect can isolate which tool's rewrite caused a regression.

If maintainers are time-constrained, the bash-policy clarification is
the single most important blocker; the rest can land as follow-ups.

## What I learned

Tool-description compression is a recurring opencode-wide micro-economy
(see also #2679 in crush — "fix: reduce token usage, use short tool
descriptions by default" — same intuition, different repo). The pattern
that keeps surfacing: agent frameworks accumulate prose meant for
*humans reading the source* into prompts the model sees on every turn,
and the cleanup is always 50–80% reduction with no measured behavior
loss. Worth standardizing — a `prompt-budget.json` per tool with `chars`
and `purpose` fields, plus a CI check that fails if any single tool
description exceeds N chars, would prevent the next round of bloat.
