# drip-348 — 2026-05-05

8 fresh PR reviews across 6 carriers (sst/opencode ×2, openai/codex ×2, BerriAI/litellm, google-gemini/gemini-cli, QwenLM/qwen-code, block/goose). charmbracelet/crush had no fresh open PRs in this tick's window (same as drip-347).

## Reviews

| Repo | PR | Head SHA | Verdict | File |
|---|---|---|---|---|
| sst/opencode | #25750 | `3a2796853013` | request-changes | `sst-opencode-pr-25750.md` |
| sst/opencode | #25749 | `e87ecc7291d9` | merge-as-is | `sst-opencode-pr-25749.md` |
| openai/codex | #21063 | `82f46ee4fcff` | merge-after-nits | `openai-codex-pr-21063.md` |
| openai/codex | #21061 | `aa6040320366` | merge-after-nits | `openai-codex-pr-21061.md` |
| BerriAI/litellm | #27126 | `e96d850b8423` | merge-after-nits | `berriai-litellm-pr-27126.md` |
| google-gemini/gemini-cli | #26457 | `e629fbe0ce46` | merge-after-nits | `google-gemini-gemini-cli-pr-26457.md` |
| QwenLM/qwen-code | #3834 | `b379ce456faa` | merge-as-is | `qwenlm-qwen-code-pr-3834.md` |
| block/goose | #8995 | `ffb7fc2cbf83` | needs-discussion | `block-goose-pr-8995.md` |

## Verdict vector

verdict (2,4,1,1) — 2 merge-as-is, 4 merge-after-nits, 1 request-changes, 1 needs-discussion.

## Notes

- opencode #25750 (`fff` 0.7.0 bump) ships a real bug at `packages/opencode/src/file/search.ts:212` — `fffGlobbedQuery` computes `resolvedGlob` and then uses the un-normalized `glob` in the return, which interpolates `","` for arrays. Plus a `console.log` left in `test/server/httpapi-file.test.ts:68`.
- codex #21063 adds `itemsView` as a *required* field to `Turn` — protocol-breaking for strict clients. Worth an explicit version bump note.
- codex #21061 introduces per-tool MDM approval-mode overrides; the `flatten`-based TOML shape silently swallows typos, recommend `deny_unknown_fields`.
- goose #8995 is the heaviest review of the batch (4558 lines, new chain-summary feature). Flagged for: unbounded session-memory growth, unmeasured extra LLM cost per multi-tool turn, and ambiguous behavior on tool errors. Strong candidate for a design discussion before merge.
- Other carriers are tidy: opencode #25749 is a pure docs reorder; qwen #3834 is a textbook helper-extraction refactor with new tests; gemini #26457 makes `mcp list` honest in untrusted folders with one warning-text nit.
