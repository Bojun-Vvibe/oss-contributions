# sst/opencode PR #25654 — fix(mcp): ensure Accept header includes both required values for Streamable HTTP

- **Link:** https://github.com/sst/opencode/pull/25654
- **Head SHA:** `bdb7d1cd852838057d58690b9c8c947c1bbfeb9e`
- **Verdict:** `merge-as-is`

## What it does

Inserts a custom `mcpFetch` wrapper at `packages/opencode/src/mcp/index.ts:12-19`
that normalizes the `Accept` header to `application/json, text/event-stream`
on every Streamable-HTTP MCP request, then passes it as the `fetch` option
to `StreamableHTTPClientTransport` at `index.ts:27`. Closes the symptom where
strict servers (Zhipu/BigModel are named in the comment, plus referenced
issues #6972 / #25650) reject the SDK's GET-side request that only carries
`Accept: text/event-stream`.

## Design analysis

- **Header normalization is the right surgical primitive.** The MCP
  Streamable-HTTP spec mandates both media types. The SDK has historically
  emitted only one for GET responses (event-stream side) and only one for
  POST bodies (json side); a server that strictly validates the union per
  spec correctly rejects either. Patching the SDK is a downstream upgrade
  away — overriding `fetch` at the transport boundary is the smallest local
  fix that covers both directions.
- **Idempotent rewrite at line 16:** the `if (!accept.includes(...) ||
  !accept.includes(...))` guard means already-correct headers pass through
  untouched and a `set("accept", ...)` only runs when needed. That avoids
  fighting the SDK if a future SDK release adds the missing media type
  upstream.
- **Headers spread via `new Headers(init?.headers)` (line 13)** correctly
  handles all three valid `HeadersInit` shapes (Headers / Record / [k,v]
  array) without losing case-sensitivity guarantees.

## Tests

The new test at `test/mcp/headers.test.ts:178-213` asserts both directions
of the property: GET-shape input (`accept: text/event-stream` only) gets
normalized, and POST-shape input (already correct) is preserved. The test
captures `globalThis.fetch` and restores it in `finally` (line 110), which
is correct for parallel-test isolation under bun test. The constructor
options shape on the mock at `headers.test.ts:46` is updated to include
`fetch?: unknown` so the new option flows through.

## Risks / nits

None blocking. Two micro-observations for the next pass:

1. `accept` lookup is case-insensitive via `Headers.get()` so the literal
   `"accept"` (lowercase) at line 14 works, but the set at line 16 also uses
   lowercase — `Headers` will canonicalize so this is fine, but a
   `const ACCEPT_HEADER = "accept"` constant would prevent typo drift.
2. No test for the `q=`-weighted accept value case (e.g.
   `application/json;q=0.9, text/event-stream;q=0.5`). The `.includes()`
   substring match would correctly accept it, but the normalization branch
   would also rewrite it because the substring check is loose. Probably the
   right tradeoff — a server strict enough to need this fix is unlikely to
   honor weighted accepts on the way back.

## What I learned

The asymmetric-failure shape (only-GET-broken, only-POST-broken depending on
the server) is exactly why the override needs to live at the *fetch* layer
and not at transport-construction time: the SDK builds a different
RequestInit per HTTP method, and a one-shot `requestInit.headers` injection
at construction time would only cover the construction-default direction.
Wrapping `fetch` is the only point downstream of all SDK call sites.
