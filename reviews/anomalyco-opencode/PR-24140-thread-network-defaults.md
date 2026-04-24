# PR #24140 — honor configured network defaults in `tui thread` mode

**Repo:** sst/opencode • **Author:** simonklee • **Status:** open
• **Net:** +2 / −2

## Change

`TuiThreadCommand` was calling `resolveNetworkOptionsNoConfig(args)`,
which intentionally bypasses config-file network defaults
(`hostname`, `port`, mDNS settings). Switch to
`resolveNetworkOptions(args)` so thread mode follows the same
resolution path as the rest of the CLI. PR description notes commit
`675a46e23e` regressed an earlier fix from `390b0a79`.

## Load-bearing risk

Two-character function-name change, but the semantics flip. The
*reason* `resolveNetworkOptionsNoConfig` exists is that some CLI
entry points must not load `TuiConfig` (typically because they run
before the config layer is available, or because they're remote-only
clients that should never bind a server). If `tui thread` is in fact
one of those — e.g. it's invoked by a remote attach flow that
shouldn't read the local user's config — this re-introduces a
different bug: the local config's `hostname`/`port` get applied to a
session that's supposed to talk to a different machine.

The PR description claims this is just restoring the pre-`675a46e23e`
behavior, but doesn't explain *why* the original commit moved away
from it. That commit may have been fixing a real downstream issue
(crash on missing config in headless thread mode? wrong port for
attach mode?) that is now silently re-broken. Worth digging into
`git log -p packages/opencode/src/cli/cmd/tui/thread.ts` for the
intent on `675a46e23e`.

A subtler concern: `resolveNetworkOptions` is now `await`ed (it
returns a Promise because it does config I/O). The surrounding
function was already async, so this is fine, but it now means thread
startup waits on disk I/O for the TUI config file. On cold
filesystems or NFS-mounted homedirs this adds latency to a
previously sync code path.

## Concrete next step

Before merging: (1) reproduce the original symptom in commit
`675a46e23e`'s message — confirm it doesn't reappear with this
change. (2) Add a test that boots `tui thread` with both
`opencode.json` containing `network.hostname: "0.0.0.0"` and a
remote-attach scenario, and assert each picks up the right value
rather than silently inheriting local config. (3) Document on the
`resolveNetworkOptions` vs `resolveNetworkOptionsNoConfig` split
*which entry points must use which*, so the next refactor doesn't
re-flip them.

## Verdict

Plausibly correct revert of an accidental regression, but the
two-symbol diff hides a real semantic question. Need the original
intent of `675a46e23e` documented before this is safe.

## What I learned

When a public API has two near-identical names (`*Foo` and
`*FooNoConfig`), every caller is one careless import-completion away
from flipping the policy. A doc-comment on each that says "DO NOT use
this from X" is cheap insurance — the alternative is a rotating
regression that ping-pongs between the two on every refactor.
