---
pr: block/goose#8913
sha: 58877aa658f3f57240dba95a345a834d38340c3e
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# feat: support google model inventory refresh

URL: https://github.com/block/goose/pull/8913
Files: `crates/goose/src/providers/google.rs`
Diff: 30+/1-

## Context

Two related fixes for Google Gemini provider parity in goose2:

1. **Inventory refresh enablement**: the Google provider now opts in
   to ACP inventory refresh by overriding `supports_inventory_refresh()
   -> true` at `google.rs:140-142` and supplying an
   `InventoryIdentityInput` at `:144-159` keyed by `GOOGLE_HOST`
   (public, defaulting to `GOOGLE_API_HOST`) and `GOOGLE_API_KEY`
   (secret, optional via `config_secret_value`). Without this, saving
   Google credentials in goose2's settings UI produced no model-list
   refresh — inconsistent with every other provider that already
   refreshes inventory on credential save.
2. **HTTP error mapping**: the `fetch_supported_models_async` path at
   `:177-184` now reads `response.status()` first, and on non-success
   maps via `map_http_error_to_provider_error(status, payload)` from
   the `openai_compatible` module. The prior code went straight to
   `response.json()` and then to `json.get("models").as_array()`,
   which on a 401/403/404 (invalid key, wrong host) silently returned
   `Some([])` from `.unwrap_or_default()` because the error JSON had
   no `models` array, producing a misleading "no models found"
   message instead of "invalid API key".

## What's good

- The `InventoryIdentityInput` at `:146-156` correctly keys the
  identity by both `GOOGLE_HOST` and `GOOGLE_API_KEY` — host so that
  switching between Generative Language API and Vertex AI endpoints
  invalidates the cached inventory, key so that swapping users
  invalidates as well.
- `with_secret("api_key", ...)` at `:155` rather than `with_public`
  ensures the API key isn't logged or surfaced in the inventory diff
  display.
- Status-first error handling at `:178-183` is the correct shape for
  any HTTP client — the prior "parse JSON, then check shape" pattern
  is a common bug class because non-JSON 502/503/504 responses
  produce `serde_json::Error` swallowed by `?`, and HTML error
  pages from an upstream proxy produce equally confusing failures.
- Reuses the existing `map_http_error_to_provider_error` helper from
  `openai_compatible`, keeping the OpenAI-style error → ProviderError
  mapping consistent across providers (so the UI surface treats a
  Google 401 the same as an OpenAI 401).
- `unwrap_or_default()` at `:182` for the body-text read is correct
  — if even the body read fails (very rare, but possible on a closed
  socket mid-response), the fallback is "report the status code with
  no payload context" rather than silently dropping the error.

## Nits / follow-ups

- The `serde_json::from_str::<serde_json::Value>(&body).ok()` at
  `:183` silently drops parse errors — for a non-JSON HTML error page
  from an intermediate proxy, the resulting `payload` is `None` and
  the user sees a status-only error with no body context. A simple
  `else` arm that surfaces the first 200 chars of the body would
  improve operator diagnostics for the corporate-proxy case.
- The optional `api_key` semantics at `:153-155` are correct for the
  case where `GOOGLE_API_KEY` is set after enrollment, but the
  identity then changes (key added → new identity → cache miss). For
  cached-inventory invalidation purposes this is correct, but worth
  documenting at the call site that "no api_key set" and "api_key
  set" are intentionally different identities.
- No test coverage added in the diff. The two changes are both
  testable: `supports_inventory_refresh()` is a const fn and trivial,
  but `inventory_identity()` reading config + the HTTP error path
  via a `mockito` server would catch future regressions cheaply
  (especially the "invalid key returns mapped error not empty list"
  case, which is the actual user-visible bug being fixed).
- `GOOGLE_PROVIDER_NAME` is reused for both arguments to
  `InventoryIdentityInput::new` at `:147` — looks like the constructor
  takes `(name, display_name)` or similar. Worth verifying that
  passing the same string twice is intentional (other providers
  presumably do the same, but a one-line confirmation in the PR body
  would be reassuring).
- The `unwrap_or_else(|_| GOOGLE_API_HOST.to_string())` at `:152`
  silently treats config errors (missing key, type mismatch) as
  "use default" — fine for the `MissingKey` variant but a `TypeMismatch`
  on `GOOGLE_HOST` should probably surface as an error rather than
  silently fall back.

## Verdict

`merge-after-nits` — both fixes are correct and address real
goose2 user-visible bugs (silent no-refresh on credential save,
misleading "no models" error on bad key). Lack of test coverage in
the diff and the silent body-parse-error dropping are the items
worth addressing before this lands or in a fast follow-up.
