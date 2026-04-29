# sst/opencode#24964 — fix(mcp): pass onprogress so resetTimeoutOnProgress actually works

- **Repo:** sst/opencode
- **PR:** [#24964](https://github.com/sst/opencode/pull/24964)
- **Head SHA:** `5d65e16bfaae076229f617c651039071e22f9e67`
- **Author:** fahreddinozcan (Context7 team)
- **Size:** +1 / -0 across 1 file (`packages/opencode/src/mcp/index.ts`)

## Summary

One-line fix in `convertMcpTool` at `packages/opencode/src/mcp/index.ts:140` — adds an
empty `onprogress: () => {}` callback to the `client.callTool` request options so that
`resetTimeoutOnProgress: true` actually does what the name says.

## What's actually going on

In the official MCP TypeScript SDK, `Protocol.request()` (and therefore
`Client.callTool`) only forwards progress notifications to the server / resets the
per-request timeout when an `onprogress` handler has been supplied by the caller. The
SDK uses presence of `onprogress` as the signal "the caller cares about progress, set
`_meta.progressToken` and wire the reset path." Without an `onprogress` callback the
`resetTimeoutOnProgress: true` flag is silently a no-op — the SDK never installs a
progress token, so the server never sends progress notifications, so the timeout
clock keeps ticking and a long-running tool gets killed at `timeout` ms even though
it's making progress.

The diff adds the minimum thing that makes the spec compliance work:

```ts
{
  onprogress: () => {},          // ← NEW
  resetTimeoutOnProgress: true,
  timeout,
},
```

An empty handler is fine here — opencode doesn't surface MCP progress events to the
user yet, so we don't need to *do* anything with the notifications, we just need to
opt into receiving them so the SDK's reset-timeout machinery engages.

## Specific line refs

- `packages/opencode/src/mcp/index.ts:140` — the new `onprogress: () => {}`.
- `packages/opencode/src/mcp/index.ts:141-142` — the existing
  `resetTimeoutOnProgress: true` and `timeout` that this fix unlocks.

## Reasoning

This is a real bug with a tiny, surgically-correct fix. Context7's report is
credible: long-running MCP tools (web fetch, large code retrieval, etc.) running over
their default timeout despite the server *trying* to send progress notifications is
exactly the failure mode an empty `onprogress` would mask. The change carries
essentially zero risk — the SDK now installs a `progressToken` in `_meta` and forwards
notifications it would otherwise drop; if a server doesn't emit progress notifications,
behavior is identical to before.

Two nits worth mentioning before merge, neither blocking:

1. The empty handler does mean opencode is now telling MCP servers "I care about
   progress" while ignoring them. That's spec-compliant but a small future-leak —
   if a server emits very frequent progress notifications, opencode pays the
   wire/decode cost for nothing. Probably not measurable in practice; worth a
   one-line `// TODO: surface progress to UI` comment so the next person knows
   why the handler is empty.
2. There's no test. A fake MCP server that emits one progress notification mid-call
   and verifies `resetTimeoutOnProgress` behavior would be ~30 lines and pin the
   regression — easy to reintroduce on a future refactor that "cleans up" the empty
   callback.

## Verdict

**merge-after-nits** — add a `// TODO: surface progress events to UI` comment at
`mcp/index.ts:140` documenting why the handler is intentionally empty (otherwise the
next refactor will delete it as dead code and silently re-break the timeout behavior),
and add a regression test using a fake MCP server that asserts a progress notification
arriving after `timeout/2` extends the deadline. Body of the change is correct and
the spec citation matches actual SDK behavior.
