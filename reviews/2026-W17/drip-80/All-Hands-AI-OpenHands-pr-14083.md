# All-Hands-AI/OpenHands PR #14083 — fix(security): redact session_api_key from WebSocket access logs

- **PR:** https://github.com/All-Hands-AI/OpenHands/pull/14083
- **Author:** all-hands-bot
- **Head SHA:** `dbad9972437b5f24e22f9b447df4c0d1c55634bb`
- **Files:** 8 (+11 / -184)
- **Verdict:** `merge-after-nits`

## What it does

Production Datadog logs from `prod-runtime/runtime-pods/runtime-*` were
exposing `session_api_key` values in plaintext via uvicorn/WebSocket
access log lines:

```
"...?resend_all=true&session_api_key=39becfd9-9a62-4507-ae37-d7fda38deaba" [accepted]
```

The fix migrates every callsite that imported from the local
`openhands.utils._redact_compat` shim to import directly from
`openhands.sdk.utils.redact` (available since SDK v1.16.1), and
**deletes the 170-line shim file**. Net diff is +11 / -184 — most of
the lines are the shim deletion.

## Specific reads

- `openhands/core/logger.py:551` — the actual log-redaction live path.
  `RedactURLParamsFilter` now imports `redact_url_params` from
  `openhands.sdk.utils.redact`. The shim's old version lived at
  `openhands/utils/_redact_compat.py` (see deleted file content) — both
  versions match in the URL-redaction logic, so this is a true
  source-swap, not a behavior change.
- `openhands/utils/_redact_compat.py` (deleted, -170 lines) — the
  shim's TODO comment said `TODO(OpenHands/evaluation#418): Delete
  this file and import directly from openhands.sdk.utils.redact once
  openhands-sdk >1.16.1 is released.` This PR is the resolution of
  that TODO. The shim contained `SENSITIVE_URL_PARAMS = frozenset({
  'tavilyapikey', 'apikey', 'api_key', 'token', 'access_token',
  'secret', 'key' })` — **`session_api_key` is NOT in this list**.
  But `_is_secret_key('session_api_key')` returns True because the
  word `KEY` appears in the upper-cased form. So the redaction does
  catch it via the secondary check, not the explicit allow-list.
- `openhands/app_server/event_callback/event_callback_models.py:24` —
  removes the `# TODO(OpenHands/evaluation#418): import from
  openhands.sdk.utils.redact` comment along with the shim import.
- `openhands/app_server/event_callback/set_title_callback_processor.py:25`
  — same pattern, swap shim import for SDK import.
- `openhands/app_server/app_conversation/live_status_app_conversation_service.py:98,109`
  — adds the SDK import, removes the shim import (the diff shows
  both — clean swap).
- `openhands/runtime/action_execution_server.py:84` — runtime side,
  same swap. **This is the file that runs inside the runtime pods
  emitting the log lines** in the PR description's Datadog example.
  So the user-visible fix lands here.
- `openhands/mcp/utils.py` — three function imports
  (`redact_text_secrets`, `redact_url_params`, `sanitize_config`) all
  swapped at once. Largest single-file import change.

## Risk surface

**Low.** Pure import-source swap with no contract change — the SDK
functions and the shim functions have identical signatures and
behavior (the shim was literally copied from the SDK with a TODO to
delete). The only ways this can regress:

1. **SDK pin mismatch.** If `openhands-sdk` isn't pinned to >=1.16.1
   in `pyproject.toml`, the imports will fail at runtime in
   environments resolving an older SDK. The PR diff doesn't show a
   `pyproject.toml` bump. Need to confirm the constraint exists, or
   add it in this PR.
2. **`_is_secret_key('session_api_key')` returning True is the
   *only* reason the bug is fixed.** If the SDK's
   `redact_url_params` ever drops the substring-check fallback in
   favor of a strict allow-list, `session_api_key` would silently
   leak again. **The right defense** is to add `session_api_key`
   explicitly to `SENSITIVE_URL_PARAMS` upstream in the SDK, so
   detection doesn't depend on the heuristic.
3. **Other call-sites still using `_redact_compat`** would break at
   import time after this PR lands (file is deleted). A `git grep
   _redact_compat` should return zero matches in this branch.

## Suggestions before merge

1. **Verify `openhands-sdk>=1.16.1` is pinned.** If not, add the
   constraint in this PR.
2. **Add `session_api_key` to `SENSITIVE_URL_PARAMS` upstream in
   the SDK.** Currently the redaction works *only* because the
   substring `KEY` is in `_is_secret_key`'s check list. That's
   fragile — if anyone refactors the SDK to use only the
   allow-list, the bug returns. Belt-and-suspenders: add it
   explicitly.
3. **Run `git grep _redact_compat`** before merge to confirm no
   stragglers.
4. **Add a regression test**: assert that
   `redact_url_params('wss://x?session_api_key=abc&other=ok')`
   returns the URL with `session_api_key` redacted but `other`
   intact. Pin the production-incident URL shape so this exact bug
   can't recur.
5. **Add a Datadog log-line scrubbing rule** as a defense-in-depth
   layer — even with this fix, *future* log paths that bypass
   `RedactURLParamsFilter` would leak. A regex scrub at the log
   ingestion side catches anything the in-process filter misses.

Verdict: merge-after-nits — the right cleanup (delete the shim, use
the SDK as canonical source) and resolves a real production
PII-in-logs incident. The two genuine concerns are the SDK pin
and the dependence on a substring heuristic to catch the offending
parameter; both are easy to address before merge.

## What I learned

URL-parameter redaction filters that rely on **substring matching
in parameter names** (`if 'KEY' in param.upper()`) are robust to new
parameters being added — `session_api_key`, `api_keys`,
`webhook_key` all match — but fragile against refactors that
replace the heuristic with an allow-list. The right defense is to
maintain *both*: an explicit allow-list of known parameters
*plus* a substring fallback for unknown future ones. And for
production secrets, you want defense in depth: in-process
redaction at log emission, *and* regex scrubbing at log ingestion.
Either one alone is one bug or one refactor away from leaking.
