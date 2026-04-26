# BerriAI/litellm PR #26549 — Fix/chatgpt gpt5.4 nonstream output

- **PR:** https://github.com/BerriAI/litellm/pull/26549
- **Head SHA:** `c5234f3db5f59bc10ab1e8525f9eff9be851f612`
- **Files:** 39 (+145 / -53)
- **Verdict:** `request-changes`

## What it does (substantive part)

In `litellm/llms/chatgpt/responses/transformation.py`, `transform_response_api_response`
now (a) accumulates `response.output_text.delta` chunk text into `output_text_parts`
and (b) when the final `response.completed` event arrives with an empty `output: []`
field but `output_text_parts` is non-empty, synthesizes a single
`{"type":"message","role":"assistant","content":[{"type":"output_text","text":"..."}]}`
entry on the `completed_response_payload`. Test
`test_chatgpt_non_stream_sse_response_parsing_empty_completed_output` at
`tests/test_litellm/llms/chatgpt/responses/test_chatgpt_responses_transformation.py:201-235`
covers the deltas-then-empty-completed shape.

## Specific reads

- `transformation.py:135-137` — adds `completed_response_payload = None` and `output_text_parts = []` accumulators. Clean.
- `transformation.py:155-159` — `OUTPUT_TEXT_DELTA` branch does `if isinstance(content_part, str) and content_part: output_text_parts.append(content_part)`; falsy/None deltas are correctly skipped. The `continue` is correct.
- `transformation.py:165` — `completed_response_payload = response_payload` is captured *after* the `dict(response_payload)` shallow copy. Good — later mutation won't leak into the original.
- `transformation.py:189-218` — fallback synthesis only fires when `not completed_response_payload.get("output") and len(output_text_parts) > 0`. Both gates are correct: skip fallback if upstream filled `output`, skip if no text accumulated.
- The `transformation.py:209` re-assigns `created_at` *again* via `_safe_convert_created_field` — but the earlier branch at `transformation.py:172` already converted it on the same payload object. Now it's converted twice. `_safe_convert_created_field` is presumably idempotent on already-converted values, but worth confirming.
- `transformation.py:215-218` — `try: ResponsesAPIResponse(**payload) except: model_construct(...)`. Bare `except` swallows `KeyboardInterrupt`/`SystemExit` — should be `except Exception`.

## Why request-changes

The transformation fix is correct and the new test is well-shaped. The blocking issue is the **PR scope**:

1. **37 unrelated files** dragged in with the fix:
   - 31× `litellm/proxy/_experimental/out/**/index.html` — static export bundle of the dashboard (`next export` output). These get regenerated on every dashboard build and should never live in a behavior-fix PR; they make the diff unreviewable and create noisy merge conflicts on every dashboard PR.
   - `ui/litellm-dashboard/package-lock.json` + `package.json` — `next` bumped from `16.1.7` → `^16.2.4`. A Next.js point-release bump and a server-side SSE parser fix have no business in the same PR. Either is fine; together they're impossible to bisect.
   - The 31 HTML files include compiled JS/CSS hashes — any reviewer trying to assess "did the dashboard change behaviorally" has to manually diff hashed bundles.
2. **Split required**:
   - **Commit A (this PR's value)**: the 80-line `transformation.py` change + the new test. Should land as one focused PR titled e.g. `fix(chatgpt-responses): synthesize output from output_text.delta when completed.output is empty`.
   - **Commit B**: the `next` minor bump alone, with a one-line note about why.
   - **Commit C**: the `_experimental/out/**` bundle regeneration. Honestly this should be a CI-managed artifact and not committed at all, but if it must be committed, it should be a maintainer's mechanical regeneration commit, not bundled with a fix.
3. **Bare `except`** at `transformation.py:215`: tighten to `except Exception` so genuine interrupts propagate.
4. **`_safe_convert_created_field` double-call** at `transformation.py:172` and `:209`: confirm idempotence in a comment, or guard with a `_already_converted` sentinel.
5. **Test gap**: add a "deltas accumulated but `completed.output` is non-empty" case asserting the upstream output is preserved verbatim and the delta-buffer is *not* substituted in. The current code is correct on that branch, but a regression there would silently double-emit assistant text.

The fix itself, isolated, is `merge-as-is`. As submitted, the 37-file unrelated payload makes me block.
