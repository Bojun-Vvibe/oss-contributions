# BerriAI/litellm #26836 — chore(mcp): encrypt user-scoped MCP credentials at rest

- **URL:** https://github.com/BerriAI/litellm/pull/26836
- **Head SHA:** `c76c300392e09c60713dd72fb1318af4db523003`
- **Files:** `litellm/proxy/_experimental/mcp_server/db.py` (+47/-49), `tests/test_litellm/proxy/_experimental/mcp_server/test_db_credentials.py` (new, +294)
- **Verdict:** `merge-as-is`

## What changed

Closes a real plaintext-at-rest leak: `LiteLLM_MCPUserCredentials.credential_b64` previously stored both BYOK API keys and OAuth2 access tokens as `base64.urlsafe_b64encode(value.encode()).decode()`, which is *encoding* not *encryption*. Anyone with read access to the DB row could `b64decode` the column back to the raw secret.

Three discrete changes:

1. **Writes go through `encrypt_value_helper`** (nacl SecretBox over `LITELLM_SALT_KEY`):
   - `store_user_credential` at `db.py:551`: `encoded = encrypt_value_helper(credential)` (was `base64.urlsafe_b64encode(credential.encode()).decode()`).
   - `store_user_oauth_credential` at `db.py:660`: `encoded = encrypt_value_helper(json.dumps(payload))` (was the same plain `urlsafe_b64encode`).

2. **Reads go through a new `_decode_user_credential` helper** at `db.py:502-521`:
   ```python
   def _decode_user_credential(stored: str) -> Optional[str]:
       decrypted = decrypt_value_helper(
           value=stored,
           key="mcp_user_credential",
           exception_type="debug",
           return_original_value=False,
       )
       if decrypted is not None:
           return decrypted
       try:
           return base64.urlsafe_b64decode(stored).decode()
       except (binascii.Error, UnicodeDecodeError, ValueError, TypeError):
           return None
   ```
   Tries nacl first (current write format); falls back to `urlsafe_b64decode` for legacy rows; returns `None` on garbage. The `exception_type="debug"` at `:507` suppresses error-log spam during the legacy fallback path — important because *every* legacy row read would otherwise log a decryption error.

3. **OAuth-payload helper `_decode_oauth_payload`** at `db.py:524-540` consolidates the three previous "decode → JSON-parse → check `type=="oauth2"`" blocks at `get_user_oauth_credential`, `list_user_oauth_credentials`, and the BYOK-overwrite guard inside `store_user_oauth_credential`. The previous code duplicated this logic three times with slightly different exception handling; the new helper unifies them and the call sites at `:691`, `:707-712`, and `:614-616` go from ~8 lines each to single calls.

The BYOK-overwrite guard at `:608-622` also gets a meaningful semantic upgrade: previously the check was "is this row a BYOK secret? raise" (exact text "A non-OAuth2 credential already exists for user ... and server ..."). New shape is "if `_decode_oauth_payload` returns `None`, refuse to overwrite" with a *substantially* better error message at `:617-622`:

```python
raise ValueError(
    f"Existing credential for user {user_id} and server "
    f"{server_id} could not be verified as an OAuth2 token. "
    f"Refusing to overwrite."
)
```

The comment at `:613-615` calls out the new failure mode explicitly: "Existing row is either a BYOK secret or an OAuth2 row that no longer decrypts (e.g. after a salt-key rotation). In either case, refuse to overwrite — the caller would clobber data that may still be recoverable." This is the right call: a salt-key rotation that breaks decrypt should *not* silently overwrite the now-unreadable OAuth row, because rotating the key back would restore access.

## Why it's right

- **Plaintext-at-rest is the real bug.** `urlsafe_b64encode` is reversible by anyone with the column value; calling it "encoding" was the unfortunate name of `credential_b64` made literal. nacl SecretBox via the existing `encrypt_value_helper` (already used elsewhere in the proxy for non-MCP secrets) is the right primitive.
- **Backward-compat is correct.** The fallback path at `:516-520` keeps existing rows readable. The `(binascii.Error, UnicodeDecodeError, ValueError, TypeError)` exception tuple at `:519` is the precise set that `urlsafe_b64decode` + `.decode()` can raise — narrower than a bare `except Exception`, which was the previous shape (`db.py:577-580` deleted), and means actual unexpected bugs surface as crashes instead of silent `None` returns.
- **`exception_type="debug"` at `:507` is load-bearing.** Without it, every legacy-row read would emit an `ERROR`-level decrypt-failure log because nacl tries-then-fails on a plain-base64 input. With it, the failure is a debug-level message and the fallback proceeds cleanly. Operators won't see a wall of false-positive errors during the migration window.
- **Round-trip and legacy tests cover the decision matrix.** The new test file at `tests/test_litellm/proxy/_experimental/mcp_server/test_db_credentials.py` (294 lines) exercises:
   - "Stored bytes ≠ plain b64 of the secret" at `:248-264` — the regression assertion. Decodes the stored value back via `urlsafe_b64decode` and asserts the secret bytes do *not* appear in the result. This is the exact form an attacker with DB read would try.
   - BYOK round-trip + legacy-row round-trip at `:267-289`.
   - OAuth round-trip + legacy-row round-trip at `:301-362` (asserts `access_token`, `refresh_token`, `scopes` all survive).
   - BYOK guard rejecting both legacy-format BYOK at `:377-380` and *encrypted* BYOK at `:383-398` — the latter being the case where the new write path runs first and a subsequent OAuth-write attempt has to refuse to clobber the now-encrypted BYOK row.
   - "Refresh path: existing OAuth payload, allow overwrite" at `:401-413` — locks the *intended* permitted overwrite.
   - `list_user_oauth_credentials` filters BYOK rows correctly at `:419-454` with three rows (encrypted OAuth, legacy OAuth, legacy BYOK) and asserts the resulting `server_ids = {"srv-encrypted", "srv-legacy"}` and `tokens = {"tok-enc", "tok-legacy"}`.
   - `_decode_user_credential("not-base64-and-not-encrypted!!!")` returns `None` at `:461-463` — the garbage-input safety net.
   - `_decode_user_credential(None)` returns `None` at `:466-468` — the defensive null-DB-value path. Without `binascii.Error` in the exception tuple this would `TypeError` on the `.decode()` call.

- **The `LITELLM_SALT_KEY` fixture at `:210-213`** uses `monkeypatch.setenv` so test runs don't pollute the real env, and the autouse fixture means every test in the file gets a stable salt. The fixed test salt (`"test-salt-key-for-byok-credential-tests-1234"`) is documented as a test-only value — no production-secret mistake risk.

## Nits / not blockers

- The two near-identical normalize-and-encrypt blocks at `db.py:551` and `:660` write through the same nacl path but with different `key=` arguments to `decrypt_value_helper` — actually wait, `encrypt_value_helper` doesn't take a `key` argument here, just the value. The `key="mcp_user_credential"` at `:507` is only on the *decrypt* call and appears to be metadata for `decrypt_value_helper`'s logging. Worth verifying the helper actually uses this for key-rotation namespacing or it's purely for log-grep.
- The `_decode_oauth_payload` `(ValueError, TypeError)` exception tuple at `:535-536` covers `json.loads` failures, but if `decoded` is somehow a `bytes` object (shouldn't happen given `_decode_user_credential` always returns `str | None`, but type-narrow defensively), `json.loads` accepts bytes too — so this is fine. Just calling out that the type contract is load-bearing.
- The fallback path at `_decode_user_credential` will *also* successfully decode plain ASCII strings that happen to be valid base64 of valid UTF-8 (which is most things). That's intentional for the legacy compatibility window, but means `_decode_user_credential(some_random_b64_string)` returns the decoded value rather than `None`. This is correct given the legacy invariant that *all* values in this column were valid `urlsafe_b64encode` output, but if the column ever held arbitrary user input, the fallback would decode garbage as if it were a real credential. Probably worth adding a one-line comment naming this assumption.
- The deletion of the `decrypt_value_helper(..., key="byok_credential", ...)` call at `db.py:577-580` (previously the fallback for nacl-encrypted rows) is correct given the new write path uses the *new* `key="mcp_user_credential"` namespace — but if there are *any* rows already encrypted under the old `key="byok_credential"` namespace from prior PRs (the diff comment "Fall back to nacl decryption for credentials stored by older code" suggests at least some), they will *not* be readable after this PR. The PR description should explicitly clarify whether `key="byok_credential"`-encrypted rows ever existed in production. If so, the fallback chain needs a third tier: nacl-with-`mcp_user_credential` → nacl-with-`byok_credential` → plain-b64.

## Risk

Low overall, but the migration cliff in nit #4 is the one to verify. If no production rows were ever written under `key="byok_credential"` (i.e. that code path was never reached because the if-try at `:572-575` always succeeded), the migration is clean. If any did exist, those rows become unreadable. The fallback to plain-b64 would still work for *those* legacy rows because they predate the nacl-encrypted-with-byok_credential branch entirely. Worth a one-line clarification in the PR description.

The encrypt-on-write change is irreversible without losing newly-stored secrets — operators rotating away from `LITELLM_SALT_KEY` after merge would be unable to read newly-stored creds, but that's the standard nacl-rotation operational discipline already in place for the rest of the proxy. The plaintext-leak fix this closes is unambiguously the higher-priority bug.
