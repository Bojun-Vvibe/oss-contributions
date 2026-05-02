# drip-273 — 2026-05-02

8 fresh OSS PR reviews across 5 upstream repos. Local artifacts only; no upstream submission.

## Reviewed PRs

| Repo | PR | Head SHA | Verdict |
|---|---|---|---|
| sst/opencode | #25403 | `af2fbc4f` | merge-after-nits |
| openai/codex | #20744 | `e7ea226b` | merge-after-nits |
| openai/codex | #20737 | `d0d7aaca` | needs-discussion |
| BerriAI/litellm | #27034 | `ee3e8121` | merge-after-nits |
| charmbracelet/crush | #2757 | `3a3b1a82` | merge-as-is |
| charmbracelet/crush | #2750 | `92b90311` | merge-as-is |
| google-gemini/gemini-cli | #26312 | `3a9b8a9d` | merge-after-nits |
| QwenLM/qwen-code | #3774 | `303b6b7d` | merge-after-nits |

## Verdict mix

- merge-as-is: 2
- merge-after-nits: 5
- needs-discussion: 1
- request-changes: 0

## Themes

- **Security-shaped fixes**: opencode #25403 (pathspec sandbox for `/undo`), litellm #27034 (config-wired URL allowlist), gemini-cli #26312 (live OAuth token refresh), qwen-code #3774 (prior-read gate before mutation).
- **Test-flake hardening**: codex #20744 (`call_id`-scoped event matching, pre-queued guardian responses).
- **Architectural consolidation**: codex #20737 (centralized approval routing — flagged needs-discussion due to size + `unreachable!()` in security-critical path), crush #2750 (score-based LSP client selection with deterministic tiebreak).
- **Provider interop**: crush #2757 (sniff image MIME to satisfy strict provider validation).
