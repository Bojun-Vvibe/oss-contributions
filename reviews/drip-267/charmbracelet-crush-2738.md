# charmbracelet/crush #2738 — fix(ui): restore API key validation + add MiniMax validation

- **Head SHA:** `bad0d43f2470be7067f6b534d3c120715a4e8c4f`
- **Files:** `AGENTS.md` (+9), `internal/config/VALIDATION.md` (+304 NEW), `internal/config/config.go` (+288 / -59), `internal/config/config_validate_test.go` (+776 NEW), `internal/ui/dialog/api_key_input.go` (+46 / -13), `internal/ui/dialog/api_key_input_test.go` (+162 NEW)
- **Verdict:** `merge-after-nits`

## Rationale

Substantive correctness fix for a real user-trust bug: the previous validation path called `GET /models` and treated `200 OK` as proof of authentication, but ~10 OpenAI-compatible gateways (AiHubMix, Avian, Cortecs, HuggingFace Router, io.net, OpenCode Go, OpenCode Zen, QiniuCloud, Synthetic, Venice — enumerated in `VALIDATION.md`) deliberately expose a public `/models`, so any string the user pasted was reported as "validated." The fix re-architects `ProviderConfig.TestConnection` in `config.go` (+288/-59) to return one of three outcomes — `nil` (verified), `ErrValidationUnsupported` (saved-not-verified, a first-class state surfaced by the UI as `APIKeyInputStateUnverified`), or any other error (invalid). The 776-line `config_validate_test.go` is essentially an audit table covering every provider's classification; the 304-line `VALIDATION.md` documents per-provider probe choices and the rule that touching `buildValidationProbe`, `classify*`, or the openai-compat allowlist requires updating the doc in the same commit. `AGENTS.md` reinforces this contract.

Nits: (1) The probe-with-malformed-body strategy ("hit `/chat/completions` with a deliberately broken payload so the server authenticates the caller before rejecting the body, without running inference") is clever but provider-dependent — please confirm none of the listed providers bill on a 4xx-rejected request, otherwise users will see line items for validation. (2) `ErrValidationUnsupported` is a sentinel, not a wrapped error — verify all comparisons use `errors.Is` rather than `==` for forward-compat. (3) The audit table should be CI-enforced (a test that fails when a new provider lands without a `VALIDATION.md` entry) — the current contract is documentation-only.

This is exactly the right shape for a security-and-trust fix: changes contract, expands tests, documents policy, and makes the un-knowable case ("saved, not verified") legible rather than faking success. Merge after billing confirmation.

