# google-gemini/gemini-cli#26234 — Allow non-https proxy urls to support container environments

- PR: https://github.com/google-gemini/gemini-cli/pull/26234
- HEAD: `1787738e045b5835b4d9c2323e1c782711b82531`
- Author: stevemk14ebr
- Files changed: 2 (+1 / -20) — `packages/core/src/core/contentGenerator.ts` (+1/-7),
  `packages/core/src/core/contentGenerator.test.ts` (-13)

## Summary

Reverts the HTTPS-required validation on custom `baseUrl` values that was
re-introduced in #25357 (which itself re-introduced behavior previously
removed in #23976). The restriction breaks valid enterprise auth-proxy
configurations in container environments where the in-cluster proxy is
reached via plain HTTP. Net delete: removes the `LOCAL_HOSTNAMES`
constant, the `url.protocol !== 'https:'` check, and the corresponding
test case asserting non-https rejection. Keeps the `try { new URL(baseUrl) }`
syntactic validation.

## Cited hunks

- `packages/core/src/core/contentGenerator.ts:113` — deletes the
  `LOCAL_HOSTNAMES = ['localhost', '127.0.0.1', '[::1]']` constant
  (now unused).
- `packages/core/src/core/contentGenerator.ts:117-126` — `validateBaseUrl`
  loses the `url: URL` capture (now `new URL(baseUrl)` is called for its
  side-effect-only `throw` behavior) and loses the
  `if (url.protocol !== 'https:' && !LOCAL_HOSTNAMES.includes(url.hostname))`
  check + its `'Custom base URL must use HTTPS unless it is localhost.'`
  error.
- `packages/core/src/core/contentGenerator.test.ts:854-863` — deletes the
  `'should reject non-https remote custom baseUrl values'` test case
  that asserted the now-removed error message.

## Risks

- This is the third trip on the same wagon (revert in #23976 → reintroduce
  in #25357 → revert in this PR). Without a regression test asserting
  *that the validation does NOT exist*, a fourth iteration is essentially
  guaranteed. The PR removes the rejection test but adds nothing
  asserting `http://example.com` is *accepted* — a one-line positive test
  would fence the policy decision against the next refactor.
- The legitimate user-protection concern that motivated #25357 (operators
  silently routing API traffic over plaintext, exposing API keys
  in-flight) is real. Reverting without a replacement signalling
  mechanism (e.g. a one-line console warning when a non-HTTPS,
  non-localhost URL is configured, controlled by a `LITELLM_INSECURE_BASEURL_OK=1`
  env var) means the next user who copy-pastes a URL with `http://` from
  a config example will silently leak their API key without realising it.
- The PR description notes a #25357 issue comment justifying the revert
  but the revert is wider than necessary — the original
  `LOCAL_HOSTNAMES` allowlist could have been extended (e.g. to private
  RFC1918 ranges, or a docker-bridge-shaped allowlist) instead of removed
  entirely. This is the minimum-viable revert; a follow-up that
  re-introduces a *configurable* HTTPS-required default (opt-out for
  container envs) is the real fix.
- No CHANGELOG entry. Operators tracking the validation policy across
  versions will be confused by the silent behavioural flip.

## Verdict

**needs-discussion**

## Recommendation

The validation restriction does break legitimate container deployments —
the revert solves a real problem. But pure revert without (a) a positive
test asserting the new acceptance contract, (b) a console-warning fallback
for the silent-plaintext-leak class, or (c) a follow-up tracking issue
for the "make this configurable" right-shape fix is going to put this PR
on the same flip-flop trajectory as #23976/#25357. Discuss whether the
right shape is "warn but allow" or "configurable strictness" rather than
landing the third unconditional flip. If consensus is to land as-is for
the container-env unblock, at minimum add the positive-acceptance test
and a CHANGELOG entry naming the policy change so the next operator
who's surprised has a paper trail.
