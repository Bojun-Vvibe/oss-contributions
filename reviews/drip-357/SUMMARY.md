# drip-357 SUMMARY (2026-05-05)

8 fresh PR reviews across 4 of 7 carriers (sst/opencode ×2, openai/codex ×3, BerriAI/litellm ×2, google-gemini/gemini-cli ×1). QwenLM/qwen-code, block/goose, and charmbracelet/crush had no fresh substantive PRs in the open-PR window — every candidate was either already in INDEX.md from prior drips or was a pure release/version/i18n/docs chore — so we doubled up on the four active carriers per the carrier-rotation rule.

## Verdict tuple

(merge-as-is, merge-after-nits, request-changes, needs-discussion) = **(1, 6, 1, 0)**

## PRs

| Repo | PR | Head SHA | Verdict | File |
|---|---|---|---|---|
| sst/opencode | #25805 | `2aec720f95b68c5cd30b8f036348f31c73e760a0` | merge-after-nits | `sst-opencode-pr-25805.md` |
| sst/opencode | #25761 | `05575d46c4057f1cf75cba8a600d11b0da3a68d5` | merge-after-nits | `sst-opencode-pr-25761.md` |
| openai/codex | #21127 | `830edd0da8d107471574fc65291cbd087184cf4d` | merge-as-is | `openai-codex-pr-21127.md` |
| openai/codex | #21120 | `a5ca8015c7d01c49693f26d3a478cbb54957cd2b` | merge-after-nits | `openai-codex-pr-21120.md` |
| openai/codex | #21109 | `09f54d7f020da76e19fc80b2e608fb0f745043e4` | merge-after-nits | `openai-codex-pr-21109.md` |
| BerriAI/litellm | #27142 | `5e52132381570b148904fab0a86d7779307ca09b` | request-changes | `berriai-litellm-pr-27142.md` |
| BerriAI/litellm | #27139 | `f1e7ee2bc17d59a23f97b6a79b77dc09bd1b9d57` | merge-after-nits | `berriai-litellm-pr-27139.md` |
| google-gemini/gemini-cli | #26480 | `6a51bcb5fa0c5103225f5b8bc92d05237fb0febc` | merge-after-nits | `google-gemini-gemini-cli-pr-26480.md` |

## Highlights

- **litellm #27142** (W3C `traceparent` → `session_id`) is a `request-changes` because the implementation uses the *whole* traceparent header (`version-traceid-spanid-flags`) as the session id instead of parsing out just the trace-id field. Two calls in the same trace get different span ids and therefore different session ids, defeating the stated "chain calls under one trace" goal. The 6 new tests on `tests/test_litellm/proxy/test_litellm_pre_call_utils.py:2279-2345` lock the wrong contract in. Also no traceparent format validation, so any garbage string is accepted.
- **codex #21127** (linux-sandbox panic-free build failure) is the cleanest PR in the batch: panic → `CodexResult` propagation → dedicated `exit_with_bwrap_build_error(err) -> !` at the binary boundary, plus a real integration test (`tests/suite/landlock.rs:590-636`) that creates a `.codex` symlink-to-decoy inside a writable workspace root and asserts both the clean error message and the absence of `"panicked at"` in stderr.
- **litellm #27139** (Vertex Agent Engine SSE iterator) correctly diagnoses that ADK emits one `finish_reason: STOP` per inner action — not per stream — and rewrites the chunk parser to surface `tool_calls` for function_call chunks, drop `finish_reason` for thought-only chunks, and only map `STOP → "stop"` when text is present. Handles both `functionCall` (REST) and `function_call` (SDK) casings. Concern: cross-chunk tool_call `index` is reset to 0 in each chunk.
- **codex #21109** (TUI `/upload` slash command) is functionally complete — wires the existing `fs/uploadFile` JSON-RPC into a slash command with proper queue-interaction test coverage — but lacks a file-size cap and tilde-expansion, so `/upload /var/log/system.log` or `/upload ~/file.txt` will OOM or fail with ENOENT respectively.
- **opencode #25805** (`max_retries` cap on session retry policy) is a strictly-additive 7-line change, but the `meta.attempt >= opts.maxRetries` semantic in `retry.ts:113` is ambiguous (does `max_retries: 3` mean "3 retries beyond the first" or "3 total attempts including first"?) and there's no unit test for the new branch.
- **codex #21120** (marketplace root removal) tightens `root.exists()` → `root.try_exists()` (so I/O errors stop being silently treated as "absent") and adds a "neither file nor directory" rejection branch. Symlink semantics still subtle: `fs::metadata` follows symlinks but `remove_dir_all`/`remove_file` operate on the link name, so a symlink-to-dir will hit the dir branch and fail with ENOTDIR.
- **opencode #25761** is a `@pierre/diffs` 1.1.0-beta.18 → 1.1.20 bump fixing the "No newline at end of file" marker false-positive. Concern: the lockfile now carries two fully-resolved copies of `shiki` (3.20.0 from `@shikijs/transformers`, 3.23.0 from the new direct `@pierre/diffs/shiki/` resolution) — bundle-size impact is unmeasured.
- **gemini-cli #26480** is a tiny prompt-tuning change for the `gemini-3` model family that fixes a real factual bug in `snippets.ts:235` (the steering text said "read_file fails if old_string is ambiguous" — `old_string` is a parameter of `replace`, not `read_file`) and adds "avoids accidental deletions" framing to the `replace` tool description. Snapshots regenerated cleanly.
