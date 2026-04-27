# BerriAI/litellm PR #26595 — OVHCloud field migration: `reasoning_content`↔`reasoning`, `duration`↔`seconds`

- **PR**: https://github.com/BerriAI/litellm/pull/26595
- **Author**: @KunalG67
- **Head SHA**: `e55e73d69b0be420a7092fcfa1159db2b7bd2d0e`
- **Size**: +153 / −4
- **Files**: `litellm/llms/ovhcloud/audio_transcription/transformation.py`, `litellm/llms/ovhcloud/chat/transformation.py`, `tests/test_litellm/llms/ovhcloud/test_ovhcloud_audio_transcription_transformation.py`, `tests/test_litellm/llms/ovhcloud/test_ovhcloud_chat_transformation.py`

## Summary

OVHCloud is renaming two response fields with a 2026-05-11 deadline: STT responses replace `duration` with `seconds`, and chat-streaming responses replace `reasoning_content` with `reasoning`. This PR teaches the litellm OVHCloud transformation layer to consume both shapes during the transition window and normalize to the legacy keys (`duration`, `reasoning_content`) so downstream litellm consumers see a consistent surface regardless of which OVHCloud server-side version handled the request.

## Verdict: `merge-after-nits`

Mostly clean migration code with three real correctness wins (zero-handling, legacy-precedence, regression tests). One minor issue: trailing-newline hygiene in two test files, and a small ergonomics question about the legacy-precedence rule for streaming.

## Specific references

- `litellm/llms/ovhcloud/audio_transcription/transformation.py:159-167` — the `seconds → duration` migration uses an explicit `is not None` check rather than truthiness:
  ```python
  duration = (
      response_json["seconds"]
      if "seconds" in response_json and response_json["seconds"] is not None
      else response_json.get("duration")
  )
  ```
  This is the right shape. A naive `response_json.get("seconds") or response_json.get("duration")` would have lost `seconds: 0.0` (genuine silence) by treating it as falsy. The test at `test_ovhcloud_audio_transcription_transformation.py:99-110` (`test_seconds_zero_mapped_to_duration`) explicitly pins this — `"seconds": 0.0` must round-trip as `_hidden_params["duration"] == 0.0`. Good.
- `litellm/llms/ovhcloud/chat/transformation.py:101-110` — the streaming `reasoning → reasoning_content` migration:
  ```python
  reasoning_new = delta.get("reasoning")
  reasoning_legacy = delta.get("reasoning_content")
  if reasoning_new is not None and reasoning_legacy is None:
      delta["reasoning_content"] = reasoning_new
  ```
  Two non-obvious choices here: (a) the new `reasoning` field is **only** copied when there is no existing `reasoning_content` — i.e. legacy wins. The test `test_streaming_both_fields_legacy_wins` at `test_ovhcloud_chat_transformation.py:359-378` pins this behavior. (b) The original code at the diff context replaced an unconditional copy (`if "reasoning" in choice["delta"]: choice["delta"]["reasoning_content"] = choice["delta"].get("reasoning")`), so the new behavior is *strictly* less aggressive — that's defensible but worth a comment about why "legacy wins" is the right rule (presumably because OVH may emit both during overlap and the legacy field is canonical until 2026-05-11).
- `tests/test_litellm/llms/ovhcloud/test_ovhcloud_audio_transcription_transformation.py:55-110` — adds three tests covering: new `seconds` → normalized `duration`, legacy `duration` still works, and `seconds == 0.0` not lost. The third test is the one that would have caught a `or`-fallback regression.
- `tests/test_litellm/llms/ovhcloud/test_ovhcloud_chat_transformation.py:295-370` — three streaming tests covering `reasoning` only, `reasoning_content` only, and both-present-legacy-wins. Each constructs a chunk dict by hand and feeds it through `OVHCloudChatCompletionStreamingHandler.chunk_parser`. The test fixture pattern is consistent with the existing tests in this file.

## Nits

1. Both new test files end with `\ No newline at end of file` (visible in the diff at audio_transcription line 110 and chat_transformation line 196). Fix the trailing newline.
2. The "legacy wins" rule for streaming (chat/transformation.py:108) deserves a clarifying comment. *Why* does legacy win? Two reasonable interpretations: (a) OVH's transition contract says servers emit both during overlap and `reasoning_content` is the source of truth — in which case "legacy wins" is correct; (b) Internal litellm code already populates `reasoning_content` and we don't want to clobber it — in which case "legacy wins" is also correct but for different reasons. Pick one and document it.
3. The audio transcription path mutates `response_json` in place (`response_json["duration"] = duration`) and then assigns it to `response._hidden_params`. This is fine because `response_json` is locally scoped, but if any future code path keeps a reference to the original dict, the mutation will leak. Defensible but worth a one-liner.
4. There's a parallel ovhcloud PR (#26595 vs the migration cluster). Consider squashing all OVHCloud field-rename PRs into one to make the 2026-05-11 deadline easier to track.
5. Minor: the chat-streaming branch at `chat/transformation.py:101` rebinds `delta = choice["delta"]` — a small readability win but it now also mutates through `delta` rather than `choice["delta"]`. Both refer to the same dict object so behavior is identical, but call this out in the diff context if a reviewer is uncertain.

## What I learned

Provider field-rename windows (legacy→new with overlap period) are a recurring source of subtle bugs in transformation layers, and the right pattern is exactly what this PR does: (1) accept both shapes, (2) normalize to one canonical shape (preferably the legacy one, so downstream code doesn't need to change yet), (3) pin the precedence rule with an explicit "both-present" test, (4) explicitly test zero/empty values to defeat truthiness traps. The `if "seconds" in response_json and response_json["seconds"] is not None` pattern is more verbose than `response_json.get("seconds")` but it correctly distinguishes "field absent" from "field present and zero" — the kind of distinction that bites silently when you trust truthiness.
