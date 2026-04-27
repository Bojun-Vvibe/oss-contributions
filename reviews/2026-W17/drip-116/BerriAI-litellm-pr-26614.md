# BerriAI/litellm#26614 — fix(anthropic): preserve reasoning content and add think-tag regression coverage

- PR: https://github.com/BerriAI/litellm/pull/26614
- Head SHA: `e5c1daf3`
- Diff: +388 / -169 across `litellm/llms/anthropic/chat/transformation.py` (+3), `litellm/llms/anthropic/experimental_pass_through/adapters/transformation.py` (large refactor mostly type-hint modernization), `litellm/llms/anthropic/experimental_pass_through/messages/handler.py`, plus tests under `tests/test_litellm/llms/anthropic/`

## What it does
Supersedes #26285 (re-raised against `litellm_internal_staging` after the previous base branch was deleted). Fixes two regressions in the Anthropic `/v1/messages` experimental pass-through path:

1. **Leaked `</think>` prefixes** could remain visible in non-streaming client text blocks. Fix at `anthropic/chat/transformation.py:1670-1671`: when `</think>` appears in `text_content`, split on first occurrence and take the right side `.lstrip()`'d. Combined with handler-level sanitization across return paths so clients always get cleaned text blocks, including a side-effect-free `_sanitize_think_tag_text_blocks()` that preserves empty post-`</think>` text blocks as `text: ""` rather than dropping them.
2. **`reasoning_content` lost in tool-call flows.** The adapter now preserves `reasoning_content` when Anthropic-compatible tool-call responses include thinking content, removes the empty-string sentinel fallback for `reasoning_content` on tool-call messages without thinking blocks (the sentinel was misrepresented downstream).

The bulk of the +388 diff is type-hint modernization in `experimental_pass_through/adapters/transformation.py` — `Dict → dict`, `List → list`, `Optional[X] → X | None`, `Union[A, B] → A | B`, plus a one-block reorder of `from openai.types.chat...` to satisfy `isort`. None of that is behavior-changing, but it inflates the line count significantly.

## Strengths
- The `</think>` strip at `transformation.py:1670-1671` is the right surgical fix — it runs *after* `reasoning_content` has already been accumulated from `thinking_content`, so the reasoning isn't lost, only the leaked closing tag is removed from the user-visible text. The `.lstrip()` after split handles the common `</think>\n\n<actual answer>` provider shape.
- The PR description explicitly notes the follow-up review fixes: (a) preserving empty post-`</think>` text blocks (so a tool-call response that's *only* thinking doesn't lose its empty-shell content block, which downstream consumers may use to detect "model finished"), and (b) making `_sanitize_think_tag_text_blocks()` side-effect-free. Both are signs the author actually iterated on review feedback — the PR is in a better state than its first revision.
- 229 tests pass on the rebuilt branch across three test files (chat transformation, adapters transformation, handler) — substantial regression coverage for both the no-thinking-block tool-call case and the thinking-only case. The author lists exact paths for the maintainer to spot-check.
- `Greptile review` checkbox is checked (Confidence Score ≥ 4/5 per pre-submission instructions), suggesting some automated cross-check has run.

## Concerns
- **Diff inflation from type-hint modernization.** The vast majority of the +388 lines in `adapters/transformation.py` are `Dict → dict`, `List → list`, `Optional[X] → X | None`, `Union → |` substitutions plus `"""Triple-quote..."""` reformatting. None of it is wrong, but it's mechanical PEP-585/PEP-604 modernization that has nothing to do with the Anthropic regression fix. It (a) makes the diff hard to review (real fix lost in noise), (b) makes blame archaeology painful, and (c) bundles a Python-version-floor implication (PEP 604 `X | None` requires 3.10+; check that litellm's stated minimum is ≥ 3.10). Recommend splitting: the bug fix lives in `anthropic/chat/transformation.py:1670-1671` plus the handler sanitizer plus the adapter's reasoning-content preservation; everything else should be a separate "chore: modernize type hints" PR.
- One Pre-Submission checkbox (`make test-unit`) remains unchecked. The PR description says "focused local validation passed" for three test files, which is good but not the same as the full unit-test sweep. A maintainer will likely ask for that.
- The `</think>` split logic at `transformation.py:1670-1671` does `text_content.split("</think>", 1)[1]` — what if the leaked tag is `< / think >` with whitespace, or `</Think>` with mixed case? Some providers emit slightly malformed closing tags. A `re.split(r'</\s*think\s*>', text_content, 1, flags=re.IGNORECASE)` would handle the common variants. Probably not needed for the providers litellm targets today, but worth flagging.
- No test in the PR description path-list specifically pins the empty-post-`</think>` text block preservation behavior. The follow-up review fix mentions it, but the test file paths listed don't single out a `test_preserves_empty_text_block_after_think_tag` case. A reader can't tell from the PR description whether the regression is pinned or just mentioned.
- The `_sanitize_think_tag_text_blocks` side-effect-free guarantee should be pinned by a test that asserts the input list is not mutated (call the function, then assert the input is `==` to a deep-copy snapshot taken before).

## Verdict
**merge-after-nits** — the actual bug fix is correct and important (leaked `</think>` tags are a visible UX regression), and the test sweep against three relevant files is solid. But the type-hint modernization should be split into a separate PR before merge so the bug fix is reviewable in isolation and bisectable later. After the split, this is clean.

## What I learned
Bundling mechanical refactors (type-hint modernization, formatting passes, import reordering) into a bug-fix PR is the most common reviewability anti-pattern in active OSS repos. The fix may be 4 lines; the diff is 388. Reviewers either (a) skim the noise and miss a real regression in the modernization, or (b) get tired and rubber-stamp. Splitting "fix" from "chore" is one of the cheapest review-quality wins available.
