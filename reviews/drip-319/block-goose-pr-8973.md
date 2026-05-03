# Review: block/goose #8973 — Improvements to LM Studio declarative provider

- PR: https://github.com/block/goose/pull/8973
- Head SHA: `28eb99893553c0a0347c584cef7312536c5a0b27`
- Author: monroewilliams
- Files touched: `crates/goose/src/providers/declarative/lmstudio.json`

## Verdict: `merge-after-nits`

## Rationale

Three sensible improvements packaged together: (1) the hard-coded `base_url` of `http://localhost:1234/v1/chat/completions` becomes `${LMSTUDIO_HOST}/v1/chat/completions`, (2) a new `env_vars` entry for `LMSTUDIO_HOST` with `required: false`, `secret: false`, `primary: true`, `default: "http://localhost:1234"`, and (3) `dynamic_models: true` so the model list is fetched at runtime instead of being statically empty. The defaults preserve current behavior for users who set nothing, which is the right call for a JSON-only change with no migration. Two nits: (1) the env var description says "Base URL of the LMStudio server (default: http://localhost:1234)" but the actual interpolation appends `/v1/chat/completions` — users supplying `LMSTUDIO_HOST=http://10.0.0.5:1234/v1` will get a malformed `/v1/v1/chat/completions` path. Either rename to `LMSTUDIO_BASE_URL` and let users provide the full prefix, or document explicitly that the value should *not* include the `/v1` segment. (2) `dynamic_models: true` with `models: []` only works if the LM Studio `/v1/models` endpoint is reliable on every supported server version; worth confirming there's a reasonable fallback in the dynamic-fetch code path so a fresh install with LM Studio offline doesn't show an empty model picker with no recovery path. Merge after the description is tightened.