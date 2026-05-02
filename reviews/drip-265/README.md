# drip-265 — 2026-05-02

8 PR reviews across 6 repos.

## Verdict tally

- merge-as-is: **2** (#20751 codex, #2750 crush)
- merge-after-nits: **5** (#25316 opencode, #20746 codex, #26991 litellm,
  #2739 crush, #3774 qwen-code)
- request-changes: **1** (#26349 gemini-cli)
- needs-discussion: **0**

## Reviews

| Repo | PR | Head SHA | Verdict | File |
|---|---|---|---|---|
| sst/opencode | #25316 | `353c7b519e253c6f9f8c8042d5550af0e533dc08` | merge-after-nits | `sst-opencode-pr-25316.md` |
| openai/codex | #20751 | `9efa5ee246fd3b456da80c75f0c3d0f8b706e0b9` | merge-as-is | `openai-codex-pr-20751.md` |
| openai/codex | #20746 | `ec74cd2ae7e451acecedc09f18ab252dbe10a952` | merge-after-nits | `openai-codex-pr-20746.md` |
| BerriAI/litellm | #26991 | `f52cbae60c259807e21af19f28b99743780b4f6c` | merge-after-nits | `berriai-litellm-pr-26991.md` |
| charmbracelet/crush | #2750 | `92b90311ecd36c82c0967e09c9777f66741145a6` | merge-as-is | `charmbracelet-crush-pr-2750.md` |
| charmbracelet/crush | #2739 | `f9b72be6420c831a35cba50fbbab1e5e2bfde4ca` | merge-after-nits | `charmbracelet-crush-pr-2739.md` |
| google-gemini/gemini-cli | #26349 | `9d8d7197f32a3f658ca0a7534d3435ab7f0a9d33` | request-changes | `google-gemini-gemini-cli-pr-26349.md` |
| QwenLM/qwen-code | #3774 | `303b6b7d4ac7706d78fc3f4e40c6bef222fa7c9d` | merge-after-nits | `qwenlm-qwen-code-pr-3774.md` |

## Themes

- **Frontmatter / config robustness:** opencode #25316 and crush #2739
  both fix silent-failure-mode bugs in config parsing — `gray-matter`
  silently dropping malformed metadata, and `jsons.Merge` carrying
  stale options across model switches.
- **LSP / tool routing correctness:** crush #2750 swaps non-deterministic
  first-match-wins for explicit scoring; qwen-code #3774 turns the
  session read-cache into an enforcement primitive that prevents
  evidence-free Edits.
- **Wire-protocol fidelity:** litellm #26991 stops dropping `effort`
  on the floor for adaptive-thinking Claude on Bedrock; codex #20751
  bounds the websocket send half so stalled writes surface promptly.
- **TUI / UX preflight:** codex #20746 catches oversized `/goal`
  before it hits the server. gemini-cli #26349 attempts the same for
  ask_user markdown but the chosen mechanism (unconditional
  unescape) is lossy — flagged for discussion.
