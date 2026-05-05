# drip-353 (2026-05-05)

## Reviews this drip

| Repo | PR | Head SHA | Verdict | File |
|---|---|---|---|---|
| sst/opencode | #25775 | `1a68eadc39772e6e51ecf3243c0c668003d98c7d` | merge-after-nits | `sst-opencode-pr-25775.md` |
| sst/opencode | #25773 | `07fa4132eb9fbd7de738bacb41743dea921aed07` | request-changes | `sst-opencode-pr-25773.md` |
| sst/opencode | #25778 | `3c3145737d57667dbdc8f03ec58b501cf73d1e6a` | request-changes | `sst-opencode-pr-25778.md` |
| openai/codex | #21111 | `5e1dbff17e658af30079558e232349033ec6b1c8` | merge-after-nits | `openai-codex-pr-21111.md` |
| openai/codex | #21107 | `c743089d83925ca7810cbc766c3b457a7188c99b` | merge-after-nits | `openai-codex-pr-21107.md` |
| BerriAI/litellm | #27143 | `b773a178cb5744e4737804cb00d33af268a8c1e4` | merge-after-nits | `berriai-litellm-pr-27143.md` |
| google-gemini/gemini-cli | #26476 | `443d046069b97e6c2edb8496cf65e813d4351048` | merge-as-is | `google-gemini-gemini-cli-pr-26476.md` |
| block/goose | #8999 | `fe16fc120255fcec47c15311fc28780d4891b2fa` | merge-after-nits | `block-goose-pr-8999.md` |

## Verdict tally

`(merge-as-is=1, merge-after-nits=5, request-changes=2, needs-discussion=0)`

## Carrier coverage

- sst/opencode ×3 (doubled up — see note below)
- openai/codex ×2 (doubled up — see note below)
- BerriAI/litellm ×1
- google-gemini/gemini-cli ×1
- block/goose ×1
- QwenLM/qwen-code: **skipped** — the 30-deep open-PR window contained zero PRs not already covered in prior drips (last fresh was #3840, already in INDEX). Doubled up on opencode instead.
- charmbracelet/crush: **skipped** — same situation, the 30-deep open-PR window is exhausted against INDEX (last fresh was #2790, already in INDEX). Doubled up on codex instead.

## Notable observations

- **litellm #27143 is a credentials-leak fix** (`secret_fields.raw_headers.authorization` was being shallow-copied into `proxy_server_request.body` and then persisted to spend logs). The fix is correct and applied at two layers (source + sanitizer) but the PR is being merged silently — flagged in the review that this should get a CVE/advisory and a "rotate keys" note.
- **opencode #25778** mixes a real cache-staleness fix with a ~50-line import-sort churn that drowns the substantive diff. Asked for the import sort to be split into its own commit before merge.
- **opencode #25773** introduces a real regression in `args.split_whitespace()` even while fixing the macOS PATH-from-Finder problem — flagged as request-changes; arguments containing spaces will be tokenized incorrectly on the new direct-spawn path.
- **codex #21107** quietly flips a config default (`trace_exporter` no longer inherits from `exporter`) — defensible (logs-endpoint trace export never worked) but needs a CHANGELOG line.
- **goose #8999** is a real safety fix (silent auto-accept on schema-less MCP elicitations is a footgun); CLI now confirms Y/n, desktop adds an Accept button — but desktop is missing a paired Decline button.
- Two carriers (qwen-code, crush) had zero fresh PRs in this window, suggesting either (a) the upstream review velocity is keeping pace with our drip cadence, or (b) we should widen the open-PR fetch beyond `--limit 30` for the next drip if we want to keep both carriers represented.
