# charmbracelet/crush PR #2798

- **Title**: fix(config): check config file for newer token before OAuth refresh
- **Author**: andreynering
- **Head SHA**: `defa17365c955a754a6dd30fe52277e18f782b22`
- **Diff**: +247 / -10

## Summary

Adds a check-disk-first path to `RefreshOAuthToken`: before hitting the provider's refresh endpoint, re-reads the on-disk config to see if another concurrent Crush session already refreshed the token. If a newer token is present, adopt it in-memory and skip the network call. Prevents wasted refresh requests + token-rotation races between sessions.

## File-by-file

### `internal/config/store.go`

- **L46-69 `RefreshOAuthToken` preamble**: new disk-first check. Logic is:
  1. `loadTokenFromDisk` (L57)
  2. on read error → log + continue to refresh (L58-59) — fail-open is correct
  3. on newer token (`AccessToken != providerConfig.OAuthToken.AccessToken`) → adopt + early return (L60-69)
- **L65-67 GH-assistant setup**: correctly threads through the `SetupGitHub...()` provider hook after token swap. Without this the in-memory provider state would be stale even though the token is fresh.
- **Concern L61**: comparison is purely string-equality on `AccessToken`. If the on-disk token has been *invalidated* (server-revoked) but is structurally newer than in-memory, we'll happily adopt it and the next API call fails. Mitigated by the network call eventually failing and triggering another refresh, but worth noting in the docstring at L44-47.
- **L72-105 refresh path**: pure rename of `newToken` → `refreshedToken` to disambiguate from the disk-load `newToken`. Mechanical and correct; verified all four downstream uses (L88, L89, L99, L100) updated.
- **L113-143 `loadTokenFromDisk`** (new):
  - Path resolution via existing `s.configPath(scope)`. Good.
  - L120-123: `os.IsNotExist` → `(nil, nil)` is correct (no config file → no token to compare → fall through to refresh).
  - L127-128: uses `gjson.Get` to extract just the `providers.<id>.oauth` subtree, then `json.Unmarshal` into `oauth.Token`. Efficient — avoids parsing the whole config.
  - L138-140: empty `AccessToken` → `(nil, nil)`. Defensive against partially-written config files. Reasonable.

### `internal/config/store_test.go` (L166-200 visible)

- `TestLoadTokenFromDisk_ReturnsNewerToken` (L166-199) sets up an on-disk config with a token and verifies all four fields round-trip (`access_token`, `refresh_token`, `expires_in`, `expires_at`). Good baseline.
- Diff is 344 lines so additional tests likely cover the negative cases (missing file, missing oauth key, empty access token, RefreshOAuthToken integration). Recommend confirming a test exists for the "newer token equals current token" no-op path.

### `go.mod` / `go.sum`

`charm.land/catwalk v0.39.3 → v0.39.5` bump bundled in. Should be called out in the PR description; bundling unrelated dep bumps in a fix PR is mildly unfortunate but the gap is small.

## Risks

- TOCTOU: another session could refresh between `loadTokenFromDisk` and the subsequent persist-back. The early-return path doesn't re-persist (it just adopts what's already on disk), so the race is harmless here. The refresh path still calls `SetConfigField` (L98-101) which itself has the usual file-write race window — not new in this PR.
- No file lock around the read. On Windows, the config file may be partially written when the other session is mid-`SetConfigField`. `gjson.Get` on a truncated JSON returns no match → safe fallback. Verified.

## Verdict

**merge-after-nits**
