# drip-304 — 2026-05-03

8 fresh PR reviews across 5 carrier projects.

## Reviews

| Repo | PR | Head SHA | Verdict | File |
|---|---|---|---|---|
| sst/opencode | #25573 | `a98026011a29` | merge-after-nits | `sst-opencode-pr-25573.md` |
| sst/opencode | #25572 | `68c427cf9ae0` | merge-as-is | `sst-opencode-pr-25572.md` |
| openai/codex | #20857 | `8d96b25122cf` | needs-discussion | `openai-codex-pr-20857.md` |
| openai/codex | #20849 | `1f9077591346` | merge-as-is | `openai-codex-pr-20849.md` |
| BerriAI/litellm | #27081 | `37a22acf6f52` | merge-after-nits | `berriai-litellm-pr-27081.md` |
| BerriAI/litellm | #27080 | `e6cdcb0188e6` | merge-after-nits | `berriai-litellm-pr-27080.md` |
| charmbracelet/crush | #2785 | `fa1acff88d05` | merge-after-nits | `charmbracelet-crush-pr-2785.md` |
| QwenLM/qwen-code | #3801 | `e0aea041ff22` | merge-after-nits | `qwenlm-qwen-code-pr-3801.md` |

## Verdict mix

- merge-as-is: 2 (#25572, #20849)
- merge-after-nits: 5 (#25573, #27081, #27080, #2785, #3801)
- request-changes: 0
- needs-discussion: 1 (#20857)

## Carriers

5 represented: sst/opencode, openai/codex, BerriAI/litellm, charmbracelet/crush, QwenLM/qwen-code.

## Themes

- **Provider-routing correctness** dominated this batch: cf-ai-gateway option keying (#25573), agentic api_base/api_key preservation (#27080), and `model_instructions_file` path resolution (#20849) are all "the existing call site silently dropped a caller-supplied parameter" bugs.
- **Security plumbing**: litellm #27081 closes a real metadata-bypass channel for observability credentials.
- **UX polish under non-interactive harnesses**: qwen-code #3801 fixes monitors disappearing from `/tasks` in headless mode; opencode #25572 adds an opt-out for click popups.
- **Protocol additions** (codex #20857) need maintainer judgment on resume/fork semantics — flagged needs-discussion rather than approving on the implementation alone.
