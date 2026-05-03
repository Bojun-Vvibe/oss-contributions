# QwenLM/qwen-code PR #3797 — feat(cli): add /model list subcommand for dynamic model discovery

- Author: B-A-M-N
- Head SHA: `9be76ee30fb99df1b62c26f38ef02c860e1815e7`
- Diff: +501 / -12 across 4 files
- Files: `packages/cli/src/ui/commands/modelCommand.test.ts`, `packages/cli/src/ui/commands/modelCommand.ts`, `packages/core/src/index.ts`, `packages/core/src/utils/fetch.ts`

## Observations

1. **`fetchModels` exports a SSRF-aware HTTP client** (proven by `modelCommand.test.ts:312-328`): rejects non-HTTPS URLs (`baseUrl must use HTTPS`), private IPs (`192.168.x.x` → `private IP`), and localhost (`localhost:8080` → `SSRF check`). For a user-facing config field that points at arbitrary remote endpoints, this is the right default posture. Confirm the SSRF check also covers `127.0.0.1`, `::1`, `169.254.x.x` (link-local / IMDS), and `10.x` / `172.16-31.x` ranges — the test only exercises `192.168.x.x` and `localhost` literally.
2. **API key sanitization in error messages** (`modelCommand.test.ts:267-278`): when the upstream returns a 500 with body `Error with secret-key-12345`, the thrown error contains `[REDACTED]` instead of the key. Critical for CLI logs that may end up in shell history or screenshots. Verify the sanitizer catches the key in *all* error paths (not just response body) — e.g. if the URL contains the key as a query param, network errors might still leak it.
3. **`fetchModels` filters non-string and empty IDs** (`modelCommand.test.ts:285-301`): `[{id: 'valid-model'}, {id: 123}, {id: ''}, {id: null}, {id: 'another-valid'}]` → `['valid-model', 'another-valid']`. Defensive against vendors that return malformed entries. Good.
4. **`modelCommand.test.ts:303-310` rejects missing `data` array**: throws `Unexpected response format: missing data array`. Correct — but consider whether the error should be more permissive for endpoints that return `{models: [...]}` instead of `{data: [...]}` (some OpenAI-compatible servers do this). If so, the error should at least suggest the alternative key in its message.
5. **Subcommand wiring** (`modelCommand.test.ts:393-449` etc.): error paths for missing config, missing content-generator config, missing baseUrl all return structured `messageType: 'error'` results rather than throwing — which is correct for a TUI command surface. The `executionMode: 'non_interactive'` setup ensures this works in scripted contexts too.
6. **Scope check — `packages/core/src/utils/fetch.ts`** is the shared fetch utility. Modifying it for one subcommand's needs deserves a quick scan: any other callers of this utility now inherit the SSRF and HTTPS-only checks. If any existing caller in core was hitting `http://` or private addresses (e.g. for local dev / testing), this PR will break them. Worth a `grep -r 'from.*utils/fetch' packages/core/src` before merging.
7. **Auth header construction** (`modelCommand.test.ts:236-251`): asserts `Authorization: Bearer <apiKey>` is set when key is provided. Standard. The test inspects the last call's `headers.Authorization` which is correct for object-form headers but might miss `Headers`-instance form — if `fetchModels` ever uses `new Headers(…)`, the test would silently pass on undefined.

## Verdict: `merge-after-nits`

The security posture (HTTPS-only, SSRF block, key redaction) is well-considered for a feature that takes user-supplied endpoints. Before merge, please: (a) extend the SSRF test cases to cover `127.0.0.1`, IPv6 localhost, and link-local/RFC1918 ranges; (b) audit other callers of `core/utils/fetch.ts` to ensure the new HTTPS-only check doesn't silently break them; (c) consider accepting `{models: […]}` as a tolerated alternative to `{data: […]}` for broader OpenAI-compatible coverage.
