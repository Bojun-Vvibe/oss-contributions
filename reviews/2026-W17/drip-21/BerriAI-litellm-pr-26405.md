# BerriAI/litellm#26405 — Fix: add fix to auth error (SSO callback OAuth-error surfacing)

- **PR:** https://github.com/BerriAI/litellm/pull/26405
- **Head SHA:** `a1275a9223d5f087e9120a7c36ac6103b5c2374d`
- **Files:** `litellm/proxy/management_endpoints/ui_sso.py` (+13/-0), `tests/test_litellm/proxy/management_endpoints/test_ui_sso.py` (+20/-0).
- **Verdict:** **merge-after-nits**

## Context

The proxy's UI SSO `auth_callback` handler at
`litellm/proxy/management_endpoints/ui_sso.py:1308` is where the
identity provider redirects after the user completes (or fails) the
auth dance. Today, when the IdP redirects with an OAuth error
response (`?error=access_denied&error_description=User%20denied%20consent`),
the handler ignores both query params and falls through to the
"verify login → load user → issue JWT" path. That path then explodes
on a missing `code` or stale `state` with an unhelpful 500 + traceback,
because none of the downstream code paths are designed to handle
"the IdP told us no".

This PR adds the missing OAuth-spec-mandated check at the top of
the handler.

## Problem

OAuth 2.0 (RFC 6749 §4.1.2.1) requires the authorization endpoint
to redirect with `error=...` (and optionally `error_description=...`,
`error_state=...`) when the resource owner denies consent or when
the request is otherwise rejected. Standard error codes include
`access_denied`, `invalid_request`, `invalid_scope`, `server_error`,
`temporarily_unavailable`, `interaction_required` (OIDC), etc.

A correctly-implemented relying party (RP) MUST inspect `error`
**before** attempting to exchange a code, because:

1. There may be no `code` to exchange — `error` and `code` are
   mutually exclusive on a redirect.
2. The user-experience requirement is to surface a sensible message
   ("you denied consent, retry?") rather than a raw 500.
3. Letting the request continue past `error` results in a noisier
   trace that hides the actual cause from the user.

Pre-PR, litellm's handler omits step 1 entirely.

## Design — what changed

A minimal 13-line block inserted at the top of `auth_callback`:

```python
oauth_error = request.query_params.get("error")
oauth_error_description = request.query_params.get("error_description")
if oauth_error:
    oauth_error_detail = f"OAuth error: {oauth_error}"
    if oauth_error_description:
        oauth_error_detail += (
            f", error_description: {oauth_error_description}"
        )
    raise HTTPException(
        status_code=400,
        detail=oauth_error_detail,
    )
```

Plus a single positive test case asserting `400` + the formatted
detail string for `error=access_denied&error_description=User denied consent`.

## Strengths

- **Right-place fix.** The check happens immediately on entry,
  before the CLI-prefix-state branch and before any DB or token-
  exchange call. There is no path past this block that can act on
  an error redirect, which is the correct invariant.
- **Status code is right.** `400 Bad Request` is the standard
  for redirect-side OAuth errors that aren't auth-failures
  per se. (`401` would also be defensible for `access_denied`
  specifically; `400` is more general.)
- **Detail string is operator-friendly.** `OAuth error: access_denied,
  error_description: User denied consent` is exactly what an SRE
  needs to triage a user complaint without having to dig through
  IdP logs.
- **Test pins the contract.** The new
  `test_auth_callback_raises_on_oauth_error` asserts both status
  and detail, so a future refactor can't silently widen or narrow
  the message.
- **Test is properly mocked.** `MagicMock(spec=Request)` constrains
  the surface so a future Starlette/FastAPI version change doesn't
  silently let the test pass on a wrong attribute name.

## Risks / nits

1. **Title is unhelpful.** "Fix: add fix to auth error" tells a
   future bisecting engineer literally nothing. Suggest:
   `fix(sso): handle IdP error redirect with explicit 400 in auth_callback`.
   This is the kind of title that gets caught in a `git log
   --oneline` scan during incident response and makes the
   difference between "I know what this commit does" and "I have
   to read the diff".
2. **`error_description` is logged into `detail` verbatim.** OAuth
   `error_description` is a free-form string supplied by the IdP.
   Most IdPs return safe ASCII, but spec-compliant ones can return
   any UTF-8. Two minor concerns:
   - **Length.** A misbehaving or hostile IdP can return an
     arbitrarily long `error_description`, which then becomes the
     entire HTTP response body. Cap at, say, 512 bytes:
     `oauth_error_description[:512]`.
   - **Log injection.** If anything downstream logs the
     `HTTPException.detail` to a structured log, embedded
     newlines or ANSI escapes from `error_description` can corrupt
     the log line. Strip control characters.
3. **No coverage for the `error` without `error_description` case.**
   The spec lists `error` as required and `error_description` as
   optional; a number of real-world IdPs (Okta, Google, etc.) return
   `error` alone in some flows. The current test only covers the
   both-present case. Add one more case asserting that
   `error=access_denied` alone yields `OAuth error: access_denied`
   without a trailing `, error_description: …`.
4. **No coverage for `error_uri`.** OAuth 2.0 also defines
   `error_uri` as an optional pointer to a human-readable error
   page. Not in scope to consume it, but the `_detail` string could
   include it for operator triage:
   `f"OAuth error: {error}{desc_part}{uri_part}"`.
5. **Other handlers in this file.** `auth_callback` isn't the only
   IdP redirect endpoint in the codebase. `ui_sso.py` and the
   surrounding `management_endpoints/` directory contain several
   adjacent handlers that may share the same omission. A grep for
   other `auth_callback`-shaped functions and a follow-up sweep
   would address the systemic issue. Out of scope for this PR
   but worth a follow-up issue.
6. **No `verbose_proxy_logger.warning(...)` on the error path.**
   The log already emits an info-level "Starting SSO callback with
   state: {state}" — adding a single warning log on the error
   branch would let operators graph IdP-error rates over time
   without depending on HTTPException-to-log propagation.

## Verdict — merge-after-nits

This is a textbook "small, correct, well-tested fix to a real spec
gap" PR. The blockers are entirely cosmetic — fix the title, cap
the description length, and add one test for the
`error`-without-`error_description` case. The actual code change
should land. There's no risk to existing happy-path SSO flows
(the new block is a no-op when `error` is absent), and the test
locks the contract in.

## What I learned

OAuth/OIDC RP implementations often grow without ever exercising
the IdP-error redirect path because in development you click
"approve" and never see the failure shape. The bug is invisible
until a real user hits "deny" or an IdP hits a maintenance window
with `temporarily_unavailable` — at which point the helpful
"OAuth error: …" 400 is the difference between a 5-minute "user
clicked deny" investigation and a 2-hour "why is the SSO endpoint
returning 500s?" outage. The right defense is: every OAuth handler
should have an `if request.query_params.get("error"):` guard at the
very top, on principle, even if you've never seen one fire. The
companion lesson is that PR titles like "Fix: add fix to auth
error" actively damage future-self productivity — `git log` becomes
the bottleneck during incident response, and self-referential
titles ("fix the fix") are the worst category of all.
