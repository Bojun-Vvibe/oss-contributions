---
pr: google-gemini/gemini-cli#25936
sha: bf3daed9edca77ec78fe274a7a88c8bad993dbd5
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(cli): filter blocked MCP servers case-insensitively

URL: https://github.com/google-gemini/gemini-cli/pull/25936
Files: `packages/cli/src/ui/components/views/McpStatus.tsx`, `packages/cli/src/ui/components/views/McpStatus.test.tsx`, snapshot
Diff: 36+/1-

## Context

`McpStatus.tsx:50-55` filters configured servers against the `blockedServers`
list to decide which servers to render as "configured" vs "blocked". The
prior implementation used strict `===` equality on `blockedServer.name ===
serverName`, which silently failed when the configured-server map key and
the blocked-server entry differed in case (`SERVER-1` vs `server-1`) or had
incidental trailing/leading whitespace from policy files / env-var splits.
A user with `SERVER-1` configured and `server-1` blocked would see the
server rendered as healthy in the status view while the runtime correctly
refused to call its tools — UI-vs-reality split-brain.

## What's good

- The fix at `McpStatus.tsx:51-54` swaps the equality predicate to
  `blockedServer.name.trim().toLowerCase() === serverName.trim().toLowerCase()`.
  Both `.trim()` and `.toLowerCase()` are applied to *both* sides of the
  comparison, so the normalization is symmetric — no risk of "we
  normalized the configured name but not the block-list name."
- Test coverage at `McpStatus.test.tsx:205-228` constructs a deliberately
  ugly scenario: configured map has keys `'SERVER-1 '` (trailing space,
  uppercase) and `' server-2'` (leading space, lowercase); blocked list
  has `' server-1'` and `'SERVER-2 '` (the symmetric mismatch). Snapshot
  output asserts both render under the "Blocked" status row — this is the
  right test shape, it would fail under the old strict-equality
  implementation.
- Snapshot file (`__snapshots__/McpStatus.test.tsx.snap:4-11`) updated
  in-tree, so reviewers can sanity-check the rendered output rather than
  trust an opaque snapshot regen.

## Nits

- The `.trim().toLowerCase()` call is now inline in two places (it'll be
  three the next time someone touches this filter). Extract to a local
  `normalizeName(name: string)` helper colocated with `McpStatus.tsx:50`,
  since the same comparison is going to be needed for `serverName ===
  selectedServer` elsewhere in the file.
- The runtime that actually gates tool calls (`mcp-client.ts` /
  registry layer) needs an audit for the same case/whitespace split — if
  that side is still strict-equality, this PR fixes the UI lie but doesn't
  fix the underlying policy matching. Worth a follow-up issue or a quick
  grep-pass to confirm.
- The configured-server map keys at `McpStatus.test.tsx:209,213` are
  arguably user-data corruption (whitespace in a config key shouldn't
  exist) — consider trimming at config-parse time too so this defensive
  matching doesn't paper over an upstream validation gap.
- No test for the `null`/`undefined` case where `blockedServer.name` is
  falsy — `.trim()` on `undefined` throws.

## Verdict reasoning

Correct, minimal, well-tested fix for a real UI-vs-policy split-brain. The
nits are about generalizing the normalization (helper, runtime parity,
config-time validation) rather than about whether to land — the patch
itself is good as-is.
