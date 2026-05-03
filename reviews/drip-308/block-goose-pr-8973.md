# block/goose #8973 — Improvements to LM Studio declarative provider

- PR: https://github.com/block/goose/pull/8973
- Author: monroewilliams
- Head SHA: `28eb99893553c0a0347c584cef7312536c5a0b27`
- Updated: 2026-05-03T03:46:20Z

## Summary
Edits `crates/goose/src/providers/declarative/lmstudio.json` so the LM Studio provider (a) reads its host from a new `LMSTUDIO_HOST` env var (default `http://localhost:1234`) instead of hardcoding the URL, and (b) sets `dynamic_models: true` to leverage LM Studio's OpenAI-compatible `/models` endpoint. Mirrors the existing `llama_swap.json` template style.

## Observations
- `crates/goose/src/providers/declarative/lmstudio.json` line ~7: `"base_url": "${LMSTUDIO_HOST}/v1/chat/completions"` — substitution syntax is consistent with peer declarative providers, but if a user sets `LMSTUDIO_HOST=http://localhost:1234/` (trailing slash, easy mistake), the resulting URL becomes `http://localhost:1234//v1/chat/completions`. Worth adding a one-line note in the `description` field warning against trailing slashes, or normalizing in the substitution layer.
- New `env_vars` block (lines ~8-16): `"required": false` + `"default": "http://localhost:1234"` is the right combo for an optional override. `"primary": true` matches `llama_swap.json`'s convention. Good consistency.
- `"dynamic_models": true` is the meaningful behavior change here — it shifts the provider from a static (empty) `models: []` list to live `/models` queries. Confirm the dynamic model loader handles the LM Studio response shape (which differs slightly from OpenAI's: LM Studio sometimes omits `created` and uses different `owned_by` values). If the loader is strict, this could break model selection for users on certain LM Studio versions.
- No test, but declarative provider JSON typically has no test surface in this repo — acceptable. A manual-test note in the PR body would have been nice though; the author does mention testing against both local and remote LM Studio instances which mitigates this.
- Single-file change to a JSON config; reverting is trivial if a user reports breakage.

## Verdict
`merge-after-nits`
