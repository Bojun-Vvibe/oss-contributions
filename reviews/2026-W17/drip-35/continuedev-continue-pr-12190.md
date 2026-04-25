# continuedev/continue #12190 — fix: use x-goog-api-key header instead of URL query param for Gemini API

- **Repo**: continuedev/continue
- **PR**: [#12190](https://github.com/continuedev/continue/pull/12190)
- **Head SHA**: `c087f99e157b61f55e4576715a4dd4d40d8ad4fb`
- **Author**: continue (bot, co-authored by bekah-hawrot-weigel)
- **State**: OPEN (+8 / -5)
- **Verdict**: `merge-as-is`

## Context

`fetchGeminiModels` in `core/llm/fetchModels.ts` was passing the
user's Google AI API key as a `?key=...` query parameter to
`https://generativelanguage.googleapis.com/v1beta/models`. That's the
classic API-key-in-URL leak pattern: keys in query strings end up in
HTTP server access logs, Sentry/Datadog request URLs, browser history
on platforms that surface fetch URLs, intermediate proxy logs, and
HTTP `Referer` headers when subsequent requests are made from the same
context.

## Design

The fix at `core/llm/fetchModels.ts:179-186`:

```diff
  const url = new URL("models", base);
- url.searchParams.set("key", apiKey ?? "");
- const response = await fetch(url);
+ const response = await fetch(url, {
+   headers: {
+     "x-goog-api-key": apiKey ?? "",
+   },
+ });
```

Two correct changes:

1. The key moves from URL query string to the `x-goog-api-key`
   request header. This is the supported and recommended Google AI
   auth path and matches what the rest of this codebase already does
   (`packages/openai-adapters/src/apis/Gemini.ts` per the PR
   description). Headers don't show up in access logs by default,
   don't propagate via `Referer`, and aren't visible in proxy URL
   capture.

2. The `?? ""` is preserved on the new code path. If `apiKey` is
   somehow undefined the request still fires (with an empty header)
   and Google returns a clean 401 instead of a TypeError. Same
   behavior as before, just on the safer transport.

The `URL("models", base)` construction is unchanged, so no risk of
URL-parsing regression.

## Companion change

`extensions/cli/src/smoke-api/smoke-api-helpers.ts` swaps the smoke-
test default model from `claude-3-haiku-20240307` (Anthropic
deprecated this; calls return `not_found_error`, breaking CI) to
`claude-haiku-4-5-20251001`. Bundling the CI fix with the security
fix is a minor scope mix, but both are tiny and the CI fix is needed
to actually run the surrounding test suite, so I'd let it ride.

## Risks / Nits

1. **No new test added.** The `fetchGeminiModels` function is
   probably exercised in some integration test, but a unit test that
   asserts no key appears in the URL string and that the
   `x-goog-api-key` header is set on the outgoing request would lock
   this regression. Mocking `fetch` and inspecting the second arg is
   ~10 lines. Worth a follow-up.

2. **No log scrubbing for prior leaks.** Existing users on metered
   monitoring (Datadog, Sentry, CloudFlare logs) may have historical
   key exposure from URL captures. Out of scope for this PR, but a
   release note suggesting "rotate your Gemini API key after
   upgrading" would be the responsible disclosure step. Worth a line
   in CHANGELOG.

3. **`apiBase` override path** — the function still honors a
   user-provided `apiBase`. If a user has configured a custom proxy
   that strips/inspects URL params (some corporate gateways do), they
   may need to allow the new header. Edge case but worth a note.

4. **Default base has trailing slash.** `apiBase ||
   "https://generativelanguage.googleapis.com/v1beta/"` — combined
   with `new URL("models", base)` the result is correct
   (`.../v1beta/models`). No change here, just confirming the URL
   stays well-formed.

## Verdict rationale

`merge-as-is`. Real security improvement, drop-in replacement, matches
the auth pattern already used elsewhere in the codebase, companion
CI-unblock fix is justified. The missing test is a nice-to-have, not a
blocker — the change is too obvious to land buggy.

## What I learned

`x-goog-api-key` is the canonical Google AI header for both Gemini and
the broader Generative Language API. URL-keyed `?key=` was the
original quickstart docs pattern and is still accepted server-side,
which is why so many client libraries got stuck on it. A repo-wide
audit (`grep -rn "key=" core/llm | grep -v x-goog`) is the obvious
follow-up to make sure no other Google-API call site still leaks via
URL.
