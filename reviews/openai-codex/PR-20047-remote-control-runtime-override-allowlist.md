# openai/codex#20047 — app-server: allow remote_control runtime feature override

- **Repo**: openai/codex
- **PR**: [#20047](https://github.com/openai/codex/pull/20047)
- **Head SHA**: `9ac525907e39e68a3f0fa689cbf1e790e81d2a0e`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-as-is`

## Context

The app-server v2 RPC `experimentalFeature/enablement/set` lets clients
patch *process-wide* runtime feature enablement, with documented
precedence:

> cloud requirements > `--enable <feature_name>` > config.toml >
> experimentalFeature/enablement/set (new) > code default

The handler in `codex-rs/app-server/src/config_api.rs` keeps an
allow-list of feature keys that are eligible for runtime override. The
`remote_control` flag has existed for a while, but wasn't on this
allow-list — so clients couldn't toggle it through the same RPC they
use for `apps`, `memories`, `plugins`, `tool_search`, `tool_suggest`,
`tool_call_mcp_elicitation`. The fix is to add it.

## What changed

### `codex-rs/app-server/src/config_api.rs:48`

```rust
const SUPPORTED_EXPERIMENTAL_FEATURE_ENABLEMENT: &[&str] = &[
    "apps",
    "memories",
    "plugins",
    "remote_control",   // ← new
    "tool_search",
    "tool_suggest",
    "tool_call_mcp_elicitation",
];
```

One added entry, alphabetically inserted. That's the entire functional
change.

### `codex-rs/app-server/README.md`

Documentation kept in sync: the `experimentalFeature/enablement/set`
bullet's "currently supported feature keys" list now reads
`apps, memories, plugins, remote_control, tool_search, tool_suggest,
tool_call_mcp_elicitation` instead of the previous (already-stale)
`apps, plugins`. Worth noting: the README was *also* missing
`memories`, `tool_search`, `tool_suggest`,
`tool_call_mcp_elicitation` — this PR is a doc bonus catching up
on multiple prior additions, not just adding `remote_control`.

### `codex-rs/app-server/tests/suite/v2/experimental_feature_list.rs`

Two test changes:

1. **New positive test**:
   `experimental_feature_enablement_set_allows_remote_control` —
   spins up `McpProcess`, calls
   `set_experimental_feature_enablement` with
   `{ "remote_control": false }`, asserts the response echoes back
   the same enablement map. End-to-end RPC coverage.

2. **Updated rejection test**: the existing
   `experimental_feature_enablement_set_rejects_non_allowlisted_feature`
   asserts the error message lists supported keys; the assertion
   string is updated to include `remote_control` in the expected
   list. This is the right shape — the test is a regression guard
   that the rejection path tells callers what *is* supported, and
   the expected list has to track the actual list.

## Risk analysis

- **Backwards-compat**: pure additive on the wire. Clients that
  previously sent `remote_control` and got rejected will now succeed.
  No client that succeeded before now fails.
- **Precedence safety**: the doc says runtime override sits below
  `--enable <feature_name>` and `config.toml`. So a deployment that
  hard-disables `remote_control` via config can't be overridden by a
  client that toggles it on at runtime. Good — this is the right
  layering for a feature with security implications.
- **Process-wide vs per-session**: the docstring says "in-memory
  process-wide" — toggling `remote_control` via this RPC affects all
  sessions on the app-server. That's the intended semantics for all
  flags on this list, but `remote_control` specifically is a
  client-mediated capability flag, so a client toggling it off
  affects every other client too. The PR doesn't change that
  semantic, but it's worth users knowing — a one-line doc note in
  the README ("note: process-wide; see precedence rules" — already
  there) covers it.
- **Concurrency**: the existing handler is documented as a "patch"
  operation. Test coverage for concurrent set calls isn't part of
  this PR; that's an existing surface, not something this PR
  introduced.

## Test gap (non-blocking)

The new test only covers `remote_control: false`. A `true` case
would also be worth running, especially if cloud-requirement
overrides apply differently for the on direction (which would be the
more dangerous direction for a remote-control toggle). Two-line
addition; happy to merge without it.

## Verdict

`merge-as-is`. Three-character functional change, doc kept in sync,
new positive test, existing negative test updated to match new
allow-list. The doc fix-up for the previously-missing flags is a
welcome side benefit. Nothing blocking.

## What I learned

When you have an explicit allow-list constant gating an RPC, the
maintenance pattern is always the same shape: feature added → array
entry → test update → README update. The interesting question for
reviewers is whether the *precedence rules* documented for the RPC
are still safe with the new key in the mix — for `remote_control`
specifically, "config.toml hard-disable wins over runtime client
toggle" is the load-bearing property. Worth making sure that
property is covered by an integration test somewhere in the suite,
even if it's not added in this PR.
