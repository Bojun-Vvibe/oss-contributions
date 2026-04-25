---
pr: 24345
repo: sst/opencode
sha: d3cb493d27af858d3b52539ca2b2744a4f55ba53
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode#24345 — fix(ripgrep): time out binary download

- **URL**: https://github.com/sst/opencode/pull/24345
- **Author**: pascalandr
- **Files**: `packages/opencode/src/file/ripgrep.ts` (+19/-2)

## Summary

The bootstrap path that downloads the bundled ripgrep binary from
GitHub releases had no client-side timeout — if the connection
stalled (captive portal, partial TLS handshake, GitHub edge slow),
the whole opencode startup hung. This PR adds a 30s timeout via
`Effect.timeoutOrElse` and consolidates the user-facing error
message so the operator gets a single actionable hint
("install ripgrep with your package manager and restart, or
retry").

## Reviewable points

- `packages/opencode/src/file/ripgrep.ts:9` introduces
  `DOWNLOAD_TIMEOUT_MS = 30_000`. 30s is the right ballpark for
  a ~3MB asset over a flaky link — short enough that a wedged
  startup doesn't look like a hang, long enough that real slow
  links (mobile tether, conference WiFi) still complete.

- `ripgrep.ts:17-22` — `downloadError()` factory builds a single
  consistent message that names the URL and suggests the package-
  manager fallback. Good: previously the error was just
  `failed to download ripgrep from <url>` (no remediation).

- `ripgrep.ts:24-27` — `normalizeDownloadError` is the careful
  bit: if the cause is *already* one of our `downloading ripgrep`
  errors (e.g. the timeout branch on line 39), don't re-wrap it
  with prefix "Failed". This avoids the timeout error being
  silently relabeled as a generic failure when it bubbles through
  `Effect.mapError` on line 41.

- `ripgrep.ts:37-40` — `Effect.timeoutOrElse` is plumbed
  *between* `response.arrayBuffer` and the `mapError` step. That
  ordering matters: on timeout we want our `downloadError(...,
  "Timed out")` to flow through `normalizeDownloadError`
  unchanged, which is exactly what the `cause.message.includes`
  guard on line 25 ensures.

- `ripgrep.ts:45` — empty-body case (zero-length 200 from a
  caching proxy or truncated response) now also goes through
  `downloadError(url, "Failed")` so the user sees the same
  actionable message.

## Risks

- Effect's `timeoutOrElse` cancels the downstream computation but
  doesn't necessarily abort the underlying HTTP request socket. On
  a Node fetch implementation, that means the socket may linger
  for kernel-level keepalive seconds after we report the timeout.
  Not a correctness issue (the result is discarded) but worth
  knowing.
- 30s is hardcoded; no env override. Fine for now — anyone behind
  a slow corporate proxy can fall back to the system package
  manager hint, which is the whole point of the new error.

## Verdict

`merge-as-is`. Tight, focused, two-line semantic change wrapped in
a clear factory; user-facing error now actionable; no test added
but the surface is a one-shot bootstrap step that's hard to
unit-test without a fake HTTP server, and the existing E2E install
path covers the happy case.

## What I learned

When you add a timeout to an effect pipeline that already has a
`mapError`, the order matters and the error-normalization function
must be idempotent over its own outputs — otherwise the carefully
worded "Timed out" string gets relabeled as "Failed" by the
downstream wrapper. The `cause.message.includes("downloading
ripgrep")` sniff is a small but important guard for that.
