# block/goose #8929 — fix(providers): refresh GCP metadata server token on expiration

- SHA: `0a6669253d7210837c9125927100a612f98eeaf4`
- State: OPEN, +34/-22 in 1 file
- File: `crates/goose/src/providers/gcpauth.rs`

## Summary

Fixes a long-lived-credential bug in the GCP ADC default-account path. Previously `AdcCredentials::DefaultAccount` cached the one-shot `TokenResponse` from the metadata server inside the variant itself, so once Goose started it kept the same access token until restart — past its `expires_in`. Patch stores the metadata `base_url` in the variant instead and makes `get_default_access_token()` re-fetch on demand, restoring the standard caching layer's expiration behavior.

## Notes

- `crates/goose/src/providers/gcpauth.rs:60-62` — `DefaultAccount(String)` with `#[serde(skip)]` is correct; you don't want a base URL serializing into a credential blob anyway. Variant signature is now `String` (base URL), much smaller surface than `TokenResponse`.
- `crates/goose/src/providers/gcpauth.rs:262-266` — drops the `.json::<TokenResponse>()` parse on initial discovery. That means the discovery call now succeeds even if the metadata server returns a malformed token at startup time — failure is deferred to the first `get_default_access_token()` call. Slight regression in fail-fast behavior, but acceptable since the original flow couldn't survive token expiry anyway.
- `crates/goose/src/providers/gcpauth.rs:534-564` — `get_default_access_token` now does a real HTTP GET against `{base_url}/computeMetadata/v1/instance/service-accounts/default/token` with the `Metadata-Flavor: Google` header. Matches the GCP metadata server contract.
- `crates/goose/src/providers/gcpauth.rs:541-552` — error mapping uses `AuthError::TokenExchange`. Consistent with the rest of the file.
- The "standard caching mechanism" referenced in the PR body presumably lives outside this function (caller-side) — should confirm there's a TTL-aware wrapper that calls `get_default_access_token` only when the cached token is near expiry. Otherwise this PR trades a never-refresh bug for a refresh-every-call performance regression.
- `crates/goose/src/providers/gcpauth.rs:1036-1037` — test updated to assert `base_url == mock_server.uri()` instead of unpacking the `TokenResponse`. Correct for the new variant. But there's no new test that exercises `get_default_access_token` itself end-to-end against `mock_server` — would catch the "is the URL constructed right" class of bug. Strongly recommend adding one.
- PR body Testing section is empty. For an auth/credential change, that's a real gap.

## Verdict

`merge-after-nits` — correct fix for a real bug, but please add: (1) a test that calls `get_default_access_token` against `mock_server` and asserts the returned `access_token` matches the metadata response, (2) confirm caller-side caching exists so this isn't fetching a fresh token on every request.
