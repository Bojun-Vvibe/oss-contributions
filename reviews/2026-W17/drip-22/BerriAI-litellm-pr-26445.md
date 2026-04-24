# PR #26445 — fix(anthropic): drop temperature from reasoning-family supported params

- URL: https://github.com/BerriAI/litellm/pull/26445
- Author: Anai-Guo
- Head SHA: `0cbe669c735e38d823f65e76a61353e07919a0de`

## Summary

`AnthropicConfig.get_supported_openai_params()` previously listed
`"temperature"` unconditionally as a supported param. Anthropic's API
rejects `temperature` for reasoning-family models (claude-3-7-sonnet,
claude-4-6, claude-opus-4-7) with `invalid_request_error: \`temperature\`
is deprecated for this model`. Because litellm believed temperature was
supported, `litellm.drop_params=True` was a no-op for these models — the
request was forwarded as-is and the upstream 400 surfaced to the caller.
Fix: move `"temperature"` out of the unconditional list and append it
only on the non-reasoning branch, leaving the existing
`thinking`/`reasoning_effort` appends on the reasoning branch unchanged.

## Specific callouts

- `litellm/llms/anthropic/chat/transformation.py` — The diff display in
  `gh pr diff` shows the entire 2,096-line file as deleted-and-re-added
  (additions: 2103 / deletions: 2096), which is a noisy presentation. The
  *substantive* change is the param-list rearrangement around the
  `is_reasoning_family` branch. Strongly recommend the author force-push a
  cleaner diff (likely a line-ending or final-newline drift caused the full
  rewrite). Reviewers cannot reliably verify a 7-line semantic change
  inside a 2,000-line diff blob — this will get an LGTM from someone who
  didn't read it line-by-line, and that's the failure mode this PR is
  itself fixing in spirit.
- The PR-body before/after snippet is the actual contract:
  ```python
  # Before
  params = ["stream", "stop", "temperature", "top_p", ...]
  if is_reasoning_family:
      params.append("thinking"); params.append("reasoning_effort")
  # After
  params = ["stream", "stop", "top_p", ...]
  if is_reasoning_family:
      params.append("thinking"); params.append("reasoning_effort")
  else:
      params.append("temperature")
  ```
  Correct shape. The `else` arm is what restores parity for non-reasoning
  models — without it, `temperature` would have been silently dropped
  everywhere, breaking standard claude-3-5-sonnet calls.
- The author's test snippet covers the right matrix: opus-4-7,
  3-7-sonnet, sonnet-4-6 should NOT report `temperature`; 3-5-sonnet,
  3-haiku should. Please commit those assertions as a real
  `tests/llm_translation/test_anthropic.py` test, not just inline in the
  PR body — otherwise the next params-list refactor has nothing to
  defend the contract.
- The reasoning-family detection helpers (`_is_opus_4_6_model`,
  `_is_opus_4_7_model`) are visible in the diff prefix. Please verify the
  branch this PR adds calls into the same helper used by
  `litellm.utils.supports_reasoning(model)` — if there are two parallel
  detectors, this fix will desync the next time a reasoning model ships.
  Use one source of truth.
- No model_prices_and_context_window.json change shown — that file is the
  canonical "is this model reasoning-family" gate for the registry-driven
  path. Confirm the fix does not need to touch
  `supports_reasoning: true` flags there.

## Risks

- Behavior change for callers who relied on `temperature` being silently
  forwarded (and presumably ignored) by these reasoning models. Anthropic
  rejects with a 400 today, so anyone doing this is already broken; but
  the *error class* changes from "upstream 400" to "param dropped before
  send". A user sending `temperature=0.0` "to be safe" will now see no
  error at all, which masks intent. A logger.info("dropping temperature
  for reasoning-family model X") is worth adding alongside the fix.
- The 2,096-line diff blob means line-by-line review effectively didn't
  happen. Reroll the PR with a clean diff before merge.

## Verdict

**Verdict:** request-changes

Fix is correct in concept. Two blocking items: (1) reroll the PR with a
clean diff (the 2096-line full-file rewrite is reviewer-hostile and hides
risk), and (2) commit the test matrix from the PR body as a real test
file. Then merge-as-is.
