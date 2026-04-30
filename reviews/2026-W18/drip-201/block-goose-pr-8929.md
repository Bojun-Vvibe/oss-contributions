# block/goose #8929 — fix(providers): refresh GCP metadata server token on expiration

- **Author:** froody (Tom Birch)
- **SHA:** `0a66692`
- **State:** OPEN
- **Size:** +34 / -22 across 1 file (`crates/goose/src/providers/gcpauth.rs`)
- **Verdict:** `merge-as-is`

## Summary

Fixes a real "long-running session breaks at the 1-hour mark" bug for GCP
metadata-server-authenticated providers. Pre-fix, `AdcCredentials::DefaultAccount`
held a `TokenResponse` (the `{token_type, access_token, expires_in}` triple) captured
once at credential-resolution time, and `get_default_access_token` at
`crates/goose/src/providers/gcpauth.rs:539+` returned `Ok(creds.clone())` — so once
the GCE/GKE metadata-server token expired (typically ~3600s), every subsequent call
got the same stale token and the upstream returned 401. Fix replaces the cached
`TokenResponse` with the metadata server's `base_url: String` at
`gcpauth.rs:60-62,260-265`, and `get_default_access_token` is rewritten at
`:533-561` to actually fetch a fresh token on each call (`GET
{base_url}/computeMetadata/v1/instance/service-accounts/default/token` with the
required `Metadata-Flavor: Google` header). The variant gets `#[serde(skip)]` at
`:60` so the on-disk-cached `AdcCredentials` shape doesn't accidentally serialize
the base URL in places where it shouldn't.

## Reasoning

The cached-token bug is exactly the failure shape "works for an hour, breaks
silently after that" which is the worst class of GCP-auth bug because it doesn't
reproduce in short test runs and it surfaces deep in the provider call stack as a
generic 401. Fetching on each call (rather than caching with a soft-refresh
heuristic) is the conservative correct fix — the metadata server is a localhost
HTTP call (`169.254.169.254` / Workload-Identity local IMDS) so per-call latency
is sub-millisecond and rate limits aren't a concern. The `#[serde(skip)]` discipline
at `:60` is the right defense against the variant accidentally being persisted
through any future `serde::Serialize` impl on `AdcCredentials` — base URLs aren't
secrets but they shouldn't end up in user-cached credential blobs either. Error
handling at `:543-554` correctly distinguishes status-non-success (typed
`AuthError::TokenExchange` with body text) from JSON-parse failure with the same
error class. The existing test at `:1036-1037` is updated to assert against the
captured `base_url` rather than the old `TokenResponse` triple, which is the right
shape — a follow-up that adds a regression for "second call fetches fresh token"
would be nice but isn't blocking since the architectural change makes it impossible
to return a stale token (the field that held the stale token no longer exists). No
banned-string surface.
