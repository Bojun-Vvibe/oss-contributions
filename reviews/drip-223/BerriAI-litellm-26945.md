# BerriAI/litellm#26945 — fix: scope CLI stored token to base_url to prevent cross-domain credential leakage

- Repo: BerriAI/litellm
- PR: https://github.com/BerriAI/litellm/pull/26945
- Head SHA: `29f2ce971194`
- Size: +72 / -14 (five files: two SDK utils, two CLI client paths, one test file)
- Verdict: **merge-as-is**

## What the diff actually does

Closes the textbook "stored credential reused on a different host" CVE shape
in the CLI/SDK login path. Five mechanically distinct moves, all coherent:

1. **`litellm/litellm_core_utils/cli_token_utils.py:34–70`** —
   `get_litellm_gateway_api_key` gains an `expected_base_url: Optional[str]
   = None` keyword. Body is rewritten from a one-liner to:
   ```python
   token_data = load_cli_token()
   if not token_data or "key" not in token_data:
       return None
   if expected_base_url is not None:
       stored_url = token_data.get("base_url")
       if stored_url != expected_base_url.rstrip("/"):
           return None
   return token_data["key"]
   ```
   The `is not None` guard preserves backwards compatibility for the
   no-argument call site (which now exists nowhere in the diff but stays
   safe for external consumers); the `rstrip("/")` is the load-bearing
   normalization that matches the storage-side normalization at the login
   path.

2. **`litellm/proxy/client/cli/commands/auth.py:56–65`** —
   `get_stored_api_key` gains the same `expected_base_url` parameter and
   forwards it to `get_litellm_gateway_api_key(expected_base_url=...)`. The
   docstring explicitly names the threat model
   ("prevents credential leakage when the CLI is pointed at a different
   (possibly malicious) server"), which is the right "future-bisecter
   discoverability" shape.

3. **`litellm/proxy/client/cli/commands/auth.py:579–586`** — `login` command
   now persists `base_url.rstrip("/")` alongside `key`/`user_id`/`user_email`
   in the saved token blob. The `rstrip` here is the symmetry-pin for the
   `rstrip` in step 1 — matched normalization at write and read is what
   makes the comparison stable across `https://x.com` vs `https://x.com/`.

4. **`litellm/proxy/client/cli/main.py:75–80`** — the cli entry point now
   passes `expected_base_url=base_url` to `get_stored_api_key`, threading
   the freshly-resolved CLI flag (or its default) into the origin check.
   This is the only call site for the CLI path and it is updated.

5. **`litellm/proxy/client/client.py:30–35`** — `Client.__init__` swaps
   `get_litellm_gateway_api_key() or api_key` for
   `api_key or get_litellm_gateway_api_key(expected_base_url=self._base_url)`
   — note both the keyword and the operand-order flip. The flip is
   load-bearing: the old code preferred the *stored* key over the
   *explicit* `api_key` argument, which inverted the principle of
   least surprise. Now an explicit `api_key` always wins, and the stored
   key only fills in when the caller didn't pass one *and* the stored
   origin matches.

6. **`tests/test_litellm/proxy/client/cli/test_auth_commands.py:234–268`**
   — adds four tests that pin the four corners of the contract:
   - `test_get_stored_api_key_base_url_match` — exact-match URL returns key
   - `test_get_stored_api_key_base_url_match_trailing_slash` — `/` is normalized
   - `test_get_stored_api_key_base_url_mismatch` — different host returns None (the CVE arm)
   - `test_get_stored_api_key_old_token_no_base_url` — pre-fix tokens without `base_url` are rejected when origin check is requested
   The fourth test is the load-bearing migration guard: it makes the
   pre-fix token blob *fail closed* when the new origin check runs,
   forcing a re-login rather than silently allowing a token whose
   origin can't be verified.

## Why merge-as-is

- **Right threat model, right fix layer.** The CVE shape is "stored
  credential designed for host A is automatically attached to a request
  to host B." Fixing it at the credential-fetch boundary
  (`get_litellm_gateway_api_key`) means every consumer (CLI command,
  Client SDK, future callers) inherits the protection without per-site
  audits. The alternative — adding origin checks at every API call site
  — would have left a long tail of unfixed paths.
- **Fail-closed for legacy tokens.** The fourth test is the most
  important one: tokens written before this PR don't carry `base_url`,
  and the new `get_litellm_gateway_api_key` returns `None` for them when
  `expected_base_url is not None`. Users get a one-time re-login prompt
  instead of a silent credential leak.
- **`rstrip("/")` symmetry at write *and* read.** Both sides normalize
  identically, so `https://proxy.com` and `https://proxy.com/` collapse
  to the same string before comparison. A future reader changing one
  side without the other will fail the trailing-slash test, which is
  the right forcing function.
- **Backwards-compatible API.** `expected_base_url` defaults to `None`,
  so external consumers of `get_litellm_gateway_api_key` keep working
  with no security regression — they just don't get the new check
  unless they opt in.
- **Operand-order flip in `client.py`** restores the principle of least
  surprise (explicit > stored) and removes the silent override that
  could surprise SDK users who explicitly passed `api_key=` and were
  getting the stored one anyway.
- **Threat-model wording in the docstring** at
  `auth.py:60–64` ("possibly malicious server") is the right level of
  blunt — future maintainers reading the function won't accidentally
  drop the parameter as "unused."

## Nits I would NOT block on

- The four-test parametrize would be a touch tighter as a single
  `@pytest.mark.parametrize` table, but four named tests with explicit
  docstrings is more grep-friendly when a future security review needs
  to find "what tests cover the origin-check arm" by name.
- No equivalent test for the `Client.__init__` operand-order flip in
  step 5 — a dedicated test pinning "explicit api_key wins over
  matching stored token" would close the symmetry, but the existing
  Client tests probably already cover the explicit-arg path indirectly.

## Theme tie-in

Fix the credential leak at the *fetch boundary* (one function, one
keyword argument), not at every read site. The same symmetric
normalization at write and read makes the comparison stable, and the
fail-closed legacy-token branch ensures no silent rollover for users
who haven't re-logged in since the fix shipped.
