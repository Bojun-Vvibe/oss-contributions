# sst/opencode #24595 — fix(opencode): don't override User-Agent set via provider options.headers

- **Repo**: sst/opencode
- **PR**: #24595
- **Author**: andocodes
- **Head SHA**: a82448915ff4adb352241e0eb4aba241e9d3302f
- **Size**: +0 / −1 in a single file:
  `packages/opencode/src/session/llm.ts`.

## What it changes

A surgical one-line deletion in the request-headers assembly
inside `live` provider layer at `packages/opencode/src/session/llm.ts:378-381`.
The previous code unconditionally injected
`"User-Agent": \`opencode/${InstallationVersion}\`` into the
default header bag (the branch that runs when the provider is
*not* the upstream API path with session-affinity headers), and
the deletion lets `...input.model.headers` and the per-call
`...headers` spread that follow override or omit it cleanly.

Spread order is preserved exactly:

```ts
{
  ...(input.model.api?... ? {...} : {
    "x-session-affinity": input.sessionID,
    ...(input.parentSessionID ? {...} : {}),
    // line 381 (old): "User-Agent": `opencode/${InstallationVersion}`,
  }),
  ...input.model.headers,
  ...headers,
}
```

## Strengths

- Right fix, right level. The previous default was inserted
  *inside* the conditional object literal, then immediately
  overridden by the two trailing spreads — except for the case
  where the user *wanted* to drop UA entirely or where the
  provider's `model.headers` was supposed to win but the user
  was reading a stack trace and seeing a default-UA header that
  they couldn't see in any config. Removing the default lets
  the spreads do what the structure already implied.
- The `...input.model.headers` and `...headers` spreads at
  `:382-383` are unchanged, so any provider config that sets
  `User-Agent` explicitly continues to win — and any provider
  config that wants the opencode default can still set it
  explicitly. Nothing else moves.
- Pure deletion (`+0/−1`). No new symbols, no behavior surface
  expansion, no test churn beyond what the existing model-
  headers tests already cover.

## Concerns / nits

- **No regression test.** The natural pin would be a one-line
  assertion in the LLM-provider-headers test: "given
  `model.headers = {}` and no caller `headers` argument, the
  outgoing header bag does not contain `User-Agent`". Without
  this, a future contributor adding a default UA back (perhaps
  citing observability) would have nothing to push back against.
  Worth one assertion before merge.
- **Operator-side observability impact unstated in the PR
  body.** Self-hosted opencode users who scrape provider
  request logs to attribute traffic to opencode versions will
  silently lose that signal after this change unless they
  configure `model.headers["User-Agent"]` themselves. Not a
  blocker — the fix is correct — but a one-line note in the
  PR body or `cli.mdx` headers section would head off support
  requests.
- The `InstallationVersion` import (if it's no longer used by
  any other branch) should be cleaned up; the diff doesn't
  show whether it has other callers in the file. Quick `rg
  InstallationVersion packages/opencode/src/session/llm.ts`
  before merge.

## Verdict

**merge-after-nits.** Right fix, wrong-shaped behavior is
gone, but a one-line regression-pin test and a one-line PR-body
note about the version-attribution side-effect would lock the
contract and head off rediscovery of the same support thread.

## What I learned

When a user-overridable config (like an HTTP header) and a
default value live in the *same object literal* with a fixed
spread order, the default essentially "wins or loses" based on
what comes after it in the spread — which is brittle because
it's positional. A cleaner pattern is to put the default in
the *base* object that gets spread *first*, so caller
overrides naturally win without anyone having to read the
order. This PR moves the code closer to that shape by
deleting the misplaced default; the next refactor could
formalize it by putting opt-in defaults in a dedicated
`defaultHeaders` object spread at position 0.
