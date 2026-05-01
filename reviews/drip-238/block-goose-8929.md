# block/goose#8929 — fix(providers): refresh GCP metadata server token on expiration

- **PR**: https://github.com/block/goose/pull/8929
- **Author**: froody (Tom Birch)
- **Head SHA**: `0a6669253d7210837c9125927100a612f98eeaf4`
- **Files**: `crates/goose/src/providers/gcpauth.rs`
- **Verdict**: **merge-after-nits**

## Context

When goose runs on a GCE/GKE instance, it pulls credentials from the GCP metadata server at `/computeMetadata/v1/instance/service-accounts/default/token`. Pre-fix, `AdcCredentials::DefaultAccount(TokenResponse)` cached the token response (access_token + token_type + expires_in) at construction time. The `get_default_access_token` impl at the prior version literally returned `Ok(creds.clone())` — meaning once the cached token expired (typically 1 hour), every subsequent provider call returned the same expired token until the process restarted. Users running long-lived goose processes on GCP would silently start failing auth ~1h in.

## What's right

- **Variant payload flipped from `TokenResponse` to `String` (base_url).** `gcpauth.rs:60-62` changes `DefaultAccount(TokenResponse)` to `#[serde(skip)] DefaultAccount(String)`. Storing the base_url instead of the token is the structurally correct fix: the credentials value now represents *how to fetch* a token (the metadata-server endpoint) rather than *the token itself*. The token's lifecycle is decoupled from the credentials' lifecycle.
- **`#[serde(skip)]` on the variant is correct.** Default-account credentials are discovered at runtime via metadata-server probe — they're never serialized to disk (unlike service-account / authorized-user creds which round-trip through ADC JSON). Skipping serde for this variant prevents accidental disk writes of the metadata-server URL.
- **Constructor at `:262-266` simplified correctly.** Pre-fix code parsed the metadata response into a `TokenResponse` and returned it cached. Post-fix code drops the body parse entirely (still status-checks the probe at `:259-261`) and returns `DefaultAccount(base_url)`. The probe is now a *liveness check* — "the metadata server responds 200, we're on a GCE instance" — rather than a *token fetch*. Right semantic.
- **`get_default_access_token` flipped from clone-cached-token to live-fetch.** `:539-565` now takes `base_url: &str`, hits `{base_url}{metadata_path}` with the load-bearing `Metadata-Flavor: Google` header (required by the metadata server, prevents SSRF-via-malicious-DNS), status-checks, parses the response, and returns a fresh `TokenResponse`. Each call gets a fresh token straight from the metadata server. The metadata server itself caches/refreshes tokens against IAM, so this isn't a hot path concern — typical metadata-server token endpoints respond in <10ms.
- **Test fixture at `:1033-1037` updated to assert the new payload shape.** `if let Ok(AdcCredentials::DefaultAccount(base_url)) = result { assert_eq!(base_url, mock_server.uri()); }` correctly verifies the constructor stores the base_url. The prior triple-assertion (`access_token` / `token_type` / `expires_in`) is gone, which is consistent with the shift — there's no token to assert at construction time anymore.
- **Error handling at `:553-562` produces actionable messages.** On non-2xx status, the code reads the response body and emits `AuthError::TokenExchange(format!("Status {}: {}", status, error_text))`. Operators debugging "why is my GCE-deployed goose suddenly failing auth" will see the exact metadata-server response.

## Risks / nits

- **No client-side token cache.** Every `get_default_access_token` call now hits the metadata server. Per-call latency is sub-10ms on GCE so this is fine in practice, but it means a burst of provider calls all serialize through the metadata-server endpoint. The metadata server itself rate-limits at thousands of QPS so this isn't a real concern, but a comment at `:539` pinning "intentionally no client-side cache; metadata server is the cache" would prevent a future contributor from "optimizing" it back to broken.
- **`response.json::<TokenResponse>()` at `:561` parses unconditionally.** If the metadata server returns 200 with an unexpected body shape (e.g. an error page from a misconfigured proxy returning HTML with 200), the JSON parse fails and the user sees `"Invalid response: <serde error>"`. This is fine for triage but if the response body is large (HTML page), the error message could be huge. Consider a body-size cap before parsing, or include only the first 200 chars of the response in the error for diagnostics.
- **No timeout on the metadata-server request.** `self.client.get(...).send().await` uses whatever default timeout the `client` has (likely none, or a long default). Metadata-server endpoints should respond in <100ms; a hung response shouldn't block goose indefinitely. Consider `.timeout(Duration::from_secs(5))` on the request.
- **The `_response.json::<TokenResponse>()` parse path was removed from the constructor at `:262-266`** — but the `TokenResponse` import is still used downstream. Confirm no dead-code warnings on `TokenResponse`'s constructor fields.
- **Test coverage gap.** I see one test asserting the new constructor shape (`:1033-1037`), but no test for the *new* live-fetch behavior on `get_default_access_token` itself — particularly the success path (`200` + valid JSON → fresh token), the non-2xx path (status check error), and the malformed-JSON path. Mock-server-backed tests for these three paths would pin the regression surface.

## Verdict

**merge-after-nits.** The structural fix is exactly right (decouple "how to fetch" from "the fetched value"; let metadata server be the cache). Wants client-side timeout, response-size-cap on error path, and three live-fetch unit tests covering success/non-2xx/malformed-JSON. The `Metadata-Flavor: Google` header preservation at `:544` is critical — confirm it never gets refactored out.
