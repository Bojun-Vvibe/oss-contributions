# BerriAI/litellm PR #27141 — Encrypt callback_vars in key/team metadata in DB

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/27141
- Head SHA: `99a124e2de03`
- Size: +454 / -6 across 10 files (1 new util, 4 endpoint patches, 1 pre-call patch, 4 test files)
- Resolves Linear LIT-1958.

## Summary

Plaintext credentials for downstream observability backends (Langfuse,
Langsmith, Arize, Braintrust, Lago, DataDog, Openmeter, AWS, GCS) were
being stored verbatim in `metadata.logging[].callback_vars` and
`metadata.callback_settings.callback_vars` on key and team rows, and
were readable through endpoints like `key/info`. This PR adds two
helpers — `encrypt_callback_vars` and `decrypt_callback_vars` — that
walk the metadata shape and encrypt/decrypt only the credential-bearing
fields, with idempotent encryption and a sentinel prefix so re-saves
through the UI don't double-wrap.

## What I like

- The sensitive-key matcher reuses the existing `SensitiveDataMasker`
  rather than maintaining a parallel allow/deny list:
  `callback_utils.py:626-630` does
  `_CALLBACK_VAR_MASKER.is_sensitive_key(key)` plus a small explicit
  set `_EXTRA_SENSITIVE_CALLBACK_KEYS = {"gcs_path_service_account"}`
  for compound names that the masker's heuristic misses. The comment
  at line 619-621 is honest about this:
  > Compound names that are credential-bearing but don't contain any
  > of the default sensitive segments (so SensitiveDataMasker won't
  > flag them).
  That's the right shape — minimize the bespoke list, lean on the
  existing infra.

- **Idempotency via sentinel prefix.** `_CALLBACK_VAR_ENCRYPTED_PREFIX
  = "litellm_enc::"` (callback_utils.py:24-27) plus the cheap prefix
  check at `_encrypt_if_plaintext` (lines 632-647) is the correct
  approach. The comment makes the salt-rotation argument explicit:
  > Cheap prefix check is robust under salt-key rotation; a
  > decrypt-based idempotency check would mis-classify K1-encrypted
  > blobs as plaintext under K2 and wrap them a second time.
  This is the trap a naive implementation would fall into. Calling
  it out in code is exactly right.

- **`copy.deepcopy(metadata)` before mutation** at
  `_transform_callback_vars` (line 588). Several test cases assert
  the input is not mutated
  (`tests/test_litellm/proxy/common_utils/test_callback_utils.py:226-232`
  — `test_encrypt_callback_vars_does_not_mutate_input`), and they
  pass because of this defensive copy. Without it, the encryption
  pass would silently rewrite caller dicts and break the
  `assert original == snapshot` invariant.

- **Both metadata shapes are handled.** The walker at
  `callback_utils.py:589-604` covers both
  `metadata["logging"][i]["callback_vars"]` and
  `metadata["callback_settings"]["callback_vars"]`. The shape is
  asymmetric (one is a list of entries, the other a single dict)
  and the code reflects that without merging the two.

- **Decrypt path is fail-open for legacy plaintext.**
  `_decrypt_or_passthrough` at lines 644-653: if value doesn't start
  with the sentinel prefix, return as-is. This is the only correct
  behavior during rollout — existing rows in production are
  plaintext, and a strict "decrypt or fail" path would brick the
  proxy on first read. The
  `test_decrypt_callback_vars_passes_through_legacy_plaintext` test
  (test_callback_utils.py:243+) locks this contract.

- **Dev-environment escape hatch.** `_encrypt_if_plaintext` lines
  640-646: if `encrypt_value_helper` raises (e.g. no
  `LITELLM_SALT_KEY` configured) the value is returned as-is, with a
  comment explaining "Dev environments without LITELLM_SALT_KEY hit
  this path; production always has a master key so encryption
  proceeds." That's pragmatic — the alternative is a 500 on key
  create in dev. Worth confirming the team is OK with the security
  posture (it does mean a misconfigured *production* proxy will
  silently store plaintext; see nit #2).

- **Pre-call decrypt is wired in three places**:
  `litellm_pre_call_utils.py:408,418,472` all wrap the metadata read
  in `decrypt_callback_vars(...)`. So the runtime path (where the
  callback actually fires) sees plaintext, while the on-disk row is
  ciphertext. That's the invariant you want.

- **Encrypt on every write surface.** All write paths get
  `encrypt_callback_vars` — `key_management_endpoints.py:1689,3181`
  (key create + key update via `prepare_metadata_fields`),
  `team_callback_endpoints.py:248,352` (add/disable team logging),
  `team_endpoints.py:1148,1815` (new_team + update_team). Any
  forgotten write path would be a leak; this looks comprehensive.

## Nits / discussion

1. **Silent-fall-through-to-plaintext is a footgun in production.**
   `_encrypt_if_plaintext` swallows *all* exceptions from
   `encrypt_value_helper` (line 645 `except Exception:`). A
   misconfigured production deploy (rotated `LITELLM_SALT_KEY`
   without proper restart, broken cryptography lib, etc.) will
   silently store plaintext credentials with no log, no metric, no
   visible signal. At minimum this should
   `verbose_proxy_logger.warning(...)` once with a stable string so
   ops can alert on it. A `LITELLM_REQUIRE_CALLBACK_VAR_ENCRYPTION`
   env var to convert this branch into a hard error in prod
   environments would also be valuable.

2. **Sentinel prefix is unauthenticated.** A user with write access
   to the metadata field could store the literal string
   `litellm_enc::not-actually-encrypted` and the decrypt path would
   try to base64-decode it, fail, and (per
   `decrypt_value_helper(..., return_original_value=False)`) return
   `None` — at which point line 653 returns the *original ciphertext
   string* as a fallback. So the callback_var ends up as the literal
   `litellm_enc::not-actually-encrypted` at runtime. Probably
   harmless (the downstream Langfuse client will just fail auth) but
   worth either: (a) HMAC the encrypted blob, or (b) make the
   "decrypt failed on a sentinel-prefixed value" path log a warning
   and return empty string instead of the raw ciphertext.

3. **Migration story for existing rows.** Existing plaintext rows
   stay plaintext until they're next written. There's no backfill
   migration. That means "encrypted at rest" is a forward-only
   property — historical credentials stay exposed in `key/info`
   responses unless every key is touched. A short README note + a
   one-shot `scripts/encrypt_existing_callback_vars.py` would close
   the loop. At minimum, the PR description / changelog should be
   explicit that this only protects newly-written rows.

4. **Per-key encryption boundary is not in the threat model.** All
   callback vars are encrypted with the same global
   `LITELLM_SALT_KEY`. A user who can read one key's row + the salt
   key can decrypt every key's callback vars. Probably acceptable
   given the "encrypt at rest" framing, but worth stating in the
   commit message so reviewers don't assume per-tenant keys.

5. **`_EXTRA_SENSITIVE_CALLBACK_KEYS = {"gcs_path_service_account"}`**
   (line 22) is a one-element set today. It will silently grow over
   time as new callbacks are added with creative key names. Worth
   either: (a) a unit test that pins the full sensitive-key set
   against a registry of known callback configurations, or (b) a
   linter that fails CI if a new callback is registered with a key
   that doesn't either match the masker heuristic or appear in this
   set. Otherwise the next "we forgot to encrypt {provider}_token"
   bug is a question of when, not if.

6. **`get_secret` is imported but unused after refactor?** Need to
   confirm — the diff at the top of `callback_utils.py` shows
   `from litellm import get_secret` still present at the existing
   import block. Just flag for the linter to confirm.

## Verdict

**merge-after-nits.** This is the right shape for fixing a real
plaintext-secrets-in-DB issue: idempotent encryption with a sentinel
prefix, deepcopy-on-write, decrypt-fail-open for legacy rows,
encryption wired into every write path, decryption wired into
every runtime read. The salt-rotation comment alone tells me the
author thought through the failure modes that matter. Before merge:
(a) replace the silent-`except Exception` in
`_encrypt_if_plaintext` with a logged warning + opt-in strict mode
for production, (b) decide whether to ship a backfill script and
document the forward-only migration in the changelog, and
(c) handle the `decrypt → None → return ciphertext` fallback at line
653 more carefully so a malformed sentinel value doesn't end up as
the raw bytes downstream. None of these block the security win.

This PR contains a real CVE-class fix (plaintext credentials in DB
exposed via `key/info`) and should ship with a security advisory
explaining: what was leaked, who could read it (anyone with key
auth), and that historical rows remain plaintext until rewritten.
