# block/goose #8929 — fix(providers): refresh GCP metadata server token on expiration

- Head SHA: `0a6669253d7210837c9125927100a612f98eeaf4`
- Files: `crates/goose/src/providers/gcpauth.rs`
- Size: +34 / -22

## What it does

Closes a real production credential bug. `AdcCredentials::DefaultAccount`
on a GCE/GKE/Cloud Run host fetches an OAuth2 access token from the
metadata server (`/computeMetadata/v1/instance/service-accounts/default/
token`), which has a finite TTL (typically 3600s). The previous
implementation cached the *entire `TokenResponse`* (including
`access_token` and `expires_in`) inside the `DefaultAccount` enum variant
itself at `gcpauth.rs:61`, and `get_default_access_token` at the old
`:541-547` simply did `Ok(creds.clone())` — meaning the access token was
treated as immortal until the Goose process restarted. After the first
3600s, every GCP API call would 401 until the user noticed and bounced
the daemon.

## The fix

The `DefaultAccount` variant payload changes from `TokenResponse` to
`String` (the metadata server base URL) at `gcpauth.rs:61`, and gains
`#[serde(skip)]` since the variant is no longer round-trippable through
serde. The construction at `gcpauth.rs:262-266` is simplified — the
metadata-server response is no longer parsed during credential
*construction*; instead the base URL is stored verbatim.
`get_default_access_token` at `gcpauth.rs:535-560` now does the actual
HTTP call on every invocation: builds the metadata path, GETs with the
`Metadata-Flavor: Google` header, surfaces non-2xx status as
`AuthError::TokenExchange`, and decodes the JSON to `TokenResponse`.

This relies on the surrounding `GcpAuth` token-cache (the "standard
caching mechanism" the body refers to) to actually decide *when* to call
`get_default_access_token` — the metadata server itself is only hit when
the cache deems the token expired. So this is not "refetch on every
request"; it's "refetch when the cache says we need a fresh token", which
is exactly the symmetric behavior of the other `AdcCredentials` variants.

## Test update

The existing test at `gcpauth.rs:1033-1041` is correctly updated: the
old assertions on `token_response.access_token` / `token_type` /
`expires_in` are gone (they no longer exist on the variant), replaced by
the simpler `assert_eq!(base_url, mock_server.uri())`. This exercises
the construction path. The actual refresh-on-expiry path is not directly
tested in the diff I see — that would require a mock metadata server
serving two distinct tokens across two `get_default_access_token` calls
plus a TTL-expiry trigger.

## What works

- The `#[serde(skip)]` on the variant is correct: a base URL captured
  at construction time isn't part of the persisted credential identity.
- `AuthError::TokenExchange` (rather than `AuthError::Credentials`) is
  the right error class for runtime-time refresh failures, matching how
  the service-account variant categorizes the same kind of HTTP error.
- The `format!("{}{}", base_url, metadata_path)` URL join at
  `gcpauth.rs:540` is safe given the `base_url` originates from the
  hard-coded `http://metadata.google.internal` constant or the test's
  `mock_server.uri()`, neither of which carries a trailing slash.

## Concerns

1. **No regression test for the actual refresh.** A two-token mock
   scenario (issue token A on first call, advance the cache's clock,
   issue token B on second call, assert the second matches B) would
   lock the bug fix. As written, the test only proves construction
   doesn't crash.
2. **Error context loss.** The previous `AuthError::Credentials`
   message ("Invalid metadata response: ...") is replaced by
   `AuthError::TokenExchange("Invalid response: ...")` at
   `gcpauth.rs:559`. Worth keeping "Invalid metadata response" in the
   message so operators can grep for the metadata-server path
   specifically vs other token-exchange failures.
3. **No retry on transient 5xx.** The metadata server is normally
   bulletproof but a single transient 5xx during refresh now
   propagates as a hard `AuthError`. A 1-2 retry with backoff would
   be a small additional safety net for production use.

The bug is real, the fix is structurally correct, the only material gap
is regression-test coverage of the actual refresh path.

Verdict: merge-after-nits
