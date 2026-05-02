# QwenLM/qwen-code PR #3799 — feat(cli): normalize model list response parsing

- Head SHA: `06e14dbf28e101fcdb37823c65684fba1b363718`
- URL: https://github.com/QwenLM/qwen-code/pull/3799
- Size: +373 / -1, 2 files (`packages/cli/src/ui/commands/modelCommand.{ts,test.ts}`)
- Verdict: **merge-after-nits**

## What changes

Adds `/model list` subcommand that fetches available models from the
configured `baseUrl + "/models"` endpoint and prints them. The bulk of
the change is the new `fetchModels(baseUrl, apiKey?)` function
(modelCommand.ts:328-385) plus 14 unit tests
(modelCommand.test.ts:194-247).

`fetchModels` accepts three response shapes:

1. Standard OpenAI: `{ data: [{ id: "..." }] }`
2. With object field (DeepSeek): `{ object: "list", data: [...] }`
3. Bare array (some providers).

Extra fields (`owned_by`, `created`, `permission`) are ignored. Empty
or whitespace-only `id` values are skipped (modelCommand.ts:380).
Trailing slashes on `baseUrl` are normalized (modelCommand.ts:331).

## What looks good

- The shape detection (modelCommand.ts:355-369) is the right
  abstraction. OpenAI-compatible `/models` is a de-facto standard but
  not a literal one — three-shape coverage is realistic and matches
  what providers actually return.
- The "skip entry with missing id" / "skip empty id" / "skip
  whitespace-only id" tests (modelCommand.test.ts:283-321) are the
  kind of defensive test you only write after you've been bitten by
  a provider returning `{ id: "  " }`. Worth keeping.
- Trailing-slash normalization
  (`baseUrl.replace(/\/+$/, '')` at modelCommand.ts:331) handles the
  `https://api.openai.com/v1/` case, which the test
  `should normalize baseUrl with trailing slash`
  (modelCommand.test.ts:355-368) verifies.
- `Authorization: Bearer ${apiKey}` is only set when `apiKey` is
  present (modelCommand.ts:340), so providers that authenticate
  via a different scheme (mTLS, query string) don't get a stray
  `Bearer undefined`.

## Nits

1. **Auth-header tests are weak.** The two
   "Authorization header when apiKey [is/is not] provided" tests
   (modelCommand.test.ts:374-403) only assert that `fetchSpy` was
   called — they don't assert *whether* the header was actually
   present. The comment on line 386 ("header verification requires
   complex typing") is a cop-out — it's just
   `expect(fetchSpy).toHaveBeenCalledWith(expect.any(String),
   expect.objectContaining({ headers: expect.objectContaining({
   Authorization: ... }) }))`. Without that, a future refactor
   that drops the header silently passes the test.
2. The import line at modelCommand.test.ts:8 has a stray space:
   `import { modelCommand , fetchModels } from './modelCommand.js';`
   — eslint/prettier should clean that up.
3. `fetchModels` swallows network errors at the top level
   (`await fetch(url, ...)` will throw on connection refused). The
   `/model list` action handler catches them at modelCommand.ts:298,
   but the test suite has no test for that path. Add one mocking
   `vi.spyOn(global, 'fetch').mockRejectedValue(new Error('ENOTFOUND'))`
   and asserting the error is surfaced.
4. The model array sort order is whatever the provider returns. For a
   long list (Together AI returns ~80 models), users will want
   alphabetic. Consider a `.sort()` before returning, or document
   that order is provider-defined.
5. The function lives in `modelCommand.ts` but is exported only "for
   testing" (per the JSDoc at modelCommand.ts:319). If it's reused
   anywhere else (e.g. config validation that pings `/models` to
   verify a key), move it to a sibling util module so the dependency
   isn't `import { fetchModels } from './ui/commands/modelCommand'`.
6. `should throw on non-object, non-array response`
   (modelCommand.test.ts:339-352) tests `'just a string'` but not
   `null` or `42`. The `body && typeof body === 'object'` guard
   (modelCommand.ts:362) handles `null` correctly, but a test
   pinning that behavior is cheap.

## Risk

Low. The new subcommand is opt-in via `/model list` and doesn't change
any existing behavior. Failure modes (bad endpoint, bad auth, weird
response shape) are routed back as `messageType: 'error'` and don't
crash the CLI. Worst case a user sees a confusing error string; the
typed exception in `fetchModels` makes that easy to refine later.
