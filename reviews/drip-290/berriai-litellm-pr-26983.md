# BerriAI/litellm PR #26983 — fix(gemini): avoid duplicate model route for full api_base

- Head SHA: `562a1452a19edbf4eb9312681b363255c75fd3aa`
- URL: https://github.com/BerriAI/litellm/pull/26983
- Size: +49 / -7, 2 files
- Verdict: **merge-after-nits**

## What changes

Fixes URL construction in `VertexBase._check_custom_proxy` for
Gemini-via-custom-`api_base`. Previously the code unconditionally did
`"{}/models/{}:{}".format(api_base, model, endpoint)`
(vertex_llm_base.py:415, removed line) which produced URLs like
`.../v1beta/models/gemini-2.5-flash-lite:generateContent/models/gemini-2.5-flash-lite:generateContent`
when the user already supplied a fully-routed `api_base`.

The new logic at vertex_llm_base.py:415-422 has three cases:

1. `api_base` already ends with `/models/{model}:{endpoint}` → use
   verbatim.
2. `api_base` ends with `/models/{model}` (no endpoint) → append
   `:{endpoint}`.
3. Otherwise → strip trailing slash and append `/models/{model}:{endpoint}`
   (also new — the old code didn't `rstrip("/")`, so a base ending in
   `/v1beta/` would have produced a `//models/...` double slash).

Tests at test_vertex_llm_base.py:892-936 parametrize all three shapes
and assert the expected URL.

## What's good

- Fixes a real misconfiguration footgun: anyone behind a proxy that
  rewrites the path (cloud reverse proxy, internal LLM gateway,
  Cloudflare worker) probably already has the full path baked into
  `api_base`. The old behavior produced a 404, with no useful error.
- The third case (`rstrip("/")`) is an unsung win — fixes the
  trailing-slash bug that was lurking in the legacy path too.
- Test parametrization covers all three branches in one place.
  Easier to maintain than separate `def test_*` functions.

## Nits

1. The `endswith` checks at vertex_llm_base.py:415-419 use the
   user-supplied `model` and `endpoint` interpolated into the suffix
   pattern. If a user passes `model="foo"` and the api_base ends with
   `/models/foobar:streamGenerateContent`, the first branch will not
   match (good) but neither will the second — they'll fall through
   to case 3 and produce
   `.../models/foobar:streamGenerateContent/models/foo:generateContent`
   (still wrong). Realistically nobody will hit this, but a
   regex-based detection (`re.search(r"/models/[^/:]+(?::\w+)?$",
   api_base)`) would be more honest and degrade gracefully.
2. The unused-import cleanup in the test file (removing
   `DEFAULT_MAX_RECURSE_DEPTH`, `litellm`, `call`) is unrelated to
   this fix. Either split into a separate cleanup commit or call it
   out in the PR description.
3. The replaced `print(f"result: {result}")` at line 71 is now
   `assert result == ("fake-token-1", "different-project")` — that's
   strictly better, but again unrelated. Same comment.
4. No test for the third case where `api_base` ends with a slash
   (`https://proxy/v1beta/`) and `endpoint=streamGenerateContent`
   *with `?alt=sse`* (streaming) — the streaming `?alt=sse` suffix
   logic is downstream and presumably unaffected, but worth a
   regression assert.

## Risk

Low/medium. The change widens the input space the function accepts
without breaking the existing one (case 3 still matches the old
behavior modulo the `rstrip`). The main risk is subtle — case 1
matching a *prefix* of someone's URL by coincidence — which is
addressed by the substring being `:{endpoint}` (not a generic
suffix).

The unused-import cleanup and assertion-instead-of-print refactor in
the same diff make the review harder than it needs to be; either
split or annotate.
