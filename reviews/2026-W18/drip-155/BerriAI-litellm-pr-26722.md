# PR #26722 — chore(tests): source live Bedrock Anthropic model from env var

- **Repo:** BerriAI/litellm
- **Link:** https://github.com/BerriAI/litellm/pull/26722
- **Author:** ryan-crabbe-berri (Ryan Crabbe)
- **State:** OPEN
- **Head SHA:** `3eee98f76ac13e99160c0a0d7c0a01412220ebe0`
- **Files:** `.circleci/config.yml` (+9/-0), `tests/local_testing/test_function_calling.py` (+9/-1), `tests/pass_through_unit_tests/test_anthropic_messages_prompt_caching.py` (+9/-2)

## Context

AWS Bedrock has been EOL'ing Anthropic Claude model variants on a rolling cadence. PR #26721 (the immediate predecessor of this PR) had to touch 16 files to swap the deprecated 3.7 Sonnet ID, and a follow-up commit there had to swap 3.5 Sonnet v2 (also EOL on 2026-03-01) for Sonnet 4.5. That's an unsustainable maintenance pattern — every EOL becomes a multi-file PR that any model-string typo would silently break.

## What changed

This PR centralises the live Bedrock Anthropic model ID behind `$BEDROCK_LIVE_ANTHROPIC_MODEL`. CircleCI's three relevant jobs (`local_testing_part1`, `local_testing_part2`, `pass_through_unit_testing`) get a job-level `environment:` block in `.circleci/config.yml` that sets the env var. The two affected test files (`test_function_calling.py` and `test_anthropic_messages_prompt_caching.py`) read the env var with a Sonnet 4.5 fallback default for local dev. Mocked transformation tests (the other ~10 files) intentionally keep their hardcoded IDs because the model string is opaque to them — it never hits AWS, so EOLs don't break them.

## Design analysis

The scope decision — only migrate live tests, leave mocked tests alone — is the right call and demonstrates the author understood the difference between "this string is data we send to AWS" and "this string is just an opaque identifier in a fixture." That separation is the whole reason this fix is small.

Two observations on the implementation that are worth pushing back on:

1. **Default fallback in code, not in CI.** The Sonnet 4.5 fallback inside `get_model()` (or whatever the parametrize helper is) means a contributor running tests locally without setting the env var hits Sonnet 4.5. That's fine *today*, but the next EOL will require updating both the CI config AND the in-code default — the very multi-file edit this PR is trying to avoid. Better to either (a) make the in-code default `None` and have the test skip with a clear message if the env var isn't set, or (b) move the default into a single shared `tests/conftest.py` or `tests/_constants.py` so future EOLs touch *one* line, not two.

2. **CircleCI env var redundancy.** Setting the same `environment:` block in three jobs invites drift — someone updates two of three and the third silently runs against an EOL'd model and fails mysteriously. CircleCI supports anchors (`&bedrock_env`, `<<: *bedrock_env`) or a context. Worth making it one source of truth.

## Risks

The `pytest --collect-only` verification in the test plan only confirms the parametrize ID resolves correctly — it doesn't run the tests. If `BEDROCK_LIVE_ANTHROPIC_MODEL` is read at module import time vs at test invocation time, a developer who exports the env var *after* importing the test module (e.g., in a Python REPL or a custom runner) might see stale values. Reading it inside the test function or via a `pytest fixture` is safer than top-level module assignment.

## Verdict

**Verdict:** merge-after-nits

Adopt YAML anchor / shared context for the CircleCI env block; collapse the in-code fallback to a single `tests/_constants.py` so the next EOL is a one-line PR. The direction is correct and the scope is well-judged.

---

*Reviewed by drip-155.*
