# block/goose#8772 — feat: ACP tool-chain metadata

- PR: https://github.com/block/goose/pull/8772
- Head SHA: `1390457c5a1255f7cbde4362271d47412b17986d`
- Author: tellaho
- Files: 9 changed, +2121 / −427

## Context

Goose's UI presents tool activity from two sources that should look identical
but didn't: the **live stream** as the agent runs, and the **replay** from
saved session bundles. The live path inferred a tool's display title from
whatever name the provider used, dropped the raw input payload after dispatch,
and grouped chained tool calls implicitly. The replay path, reconstructing
from persisted ACP notifications, lacked the metadata to recover the same
title or the same chain grouping — so refreshed sessions looked worse than
the original run, and any "grouped tool card" UX feature would build on
quicksand because live and replay produced incompatible UI state.

This PR makes the ACP server emit explicit, deterministic tool-chain
metadata and titles, then wires Goose2's UI handlers to consume that
metadata so live and replay produce byte-equivalent UI state.

## Design

Three layers move in lockstep.

**Layer 1: ACP server emits structured chain metadata
(`crates/goose/src/acp/server.rs:139-470`).** Two new constants and a
helper:

```rust
fn tool_chain_meta(chain_id: &str, chain_summary: &str) -> AcpMeta {
    let mut meta = AcpMeta::new();
    meta.insert(
        TOOL_CHAIN_META_KEY.to_string(),
        serde_json::Value::String(chain_id.to_string()),
    );
    meta.insert(
        TOOL_CHAIN_SUMMARY_META_KEY.to_string(),
        serde_json::Value::String(chain_summary.to_string()),
    );
    meta
}
```

Plus a substantial set of ACP-↔-RMCP conversion helpers (`acp_meta_to_rmcp`,
`rmcp_meta_to_acp`, `acp_annotations_to_rmcp`, `rmcp_annotations_to_acp`,
`acp_text_to_message_content`, etc., lines ~115-470). The conversions are
direction-symmetric — important because the server now needs to hand off
metadata in both directions across the ACP/RMCP boundary, and any
asymmetry would cause "live used the rich shape, replay reconstructed a
lossy projection."

The annotations conversion correctly handles the `Role` enum mismatch by
mapping unknown ACP roles to `Role::User` rather than panicking
(`server.rs:139-148`):
```rust
.map(|role| match role {
    AcpRole::Assistant => Role::Assistant,
    AcpRole::User => Role::User,
    _ => Role::User,
})
```
That's the right defensive default for a forward-compatible enum, though
worth a `tracing::warn!` to surface unknown roles instead of silently
collapsing them.

**Layer 2: ACP notification handler in Goose2 consumes the new metadata
(`ui/goose2/src/shared/api/acpNotificationHandler.ts:181-308`).** The
replay path's `tool_call` and `tool_call_update` handlers go from this
shape (pre-PR):

```ts
case "tool_call": {
  msg.content.push({
    type: "toolRequest",
    id: update.toolCallId,
    name: update.title,        // just the title
    arguments: {},             // dropped
    status: "executing",
    startedAt: Date.now(),
  });
}
```

to (post-PR):

```ts
case "tool_call": {
  const payload = normalizeToolCallPayload(update);
  msg.content.push(buildToolRequestBlock(update, payload));
}
```

The `normalizeToolCallPayload` + `buildToolRequestBlock` helpers (in the
new `toolCallChainMeta.ts` and `toolCallBlockBuilders.ts`) preserve the
raw input arguments, the chain id/summary, and the canonical tool name —
which is what makes replay match live.

The `tool_call_update` rewrite is similarly principled: the previous code
mutated the toolRequest in-place with `(tc as ToolRequestContent).name =
update.title` and pushed a separately-constructed `toolResponse` referring
back to `tc.name`. The new code uses immutable `updateToolCallBlocks(...)`
with a structured `buildToolCallPatch(...)` payload, then reads the
updated request via `findToolRequestById` for the response block. That's
both safer (no in-place mutation of an array element while iterating its
neighbors) and necessary for the live/replay-parity goal — both paths now
go through the same builders.

**Layer 3: Tool-chain metadata helpers
(`ui/goose2/src/shared/api/toolCallChainMeta.ts`, new).** Centralizes
chain-id extraction and summary derivation. Used by both live and replay
handlers, so the chain grouping computed at stream time is the same
chain grouping computed at replay time.

Coverage is added in `acpNotificationHandler.test.ts` (379 new lines of
tests, line 78 onwards) — a substantial test addition that exercises the
live/replay parity claim with concrete fixtures.

## Risk analysis

**Surface area.** 9 files, +2121 / −427 = the largest diff in the drip.
The bulk lands in `crates/goose/src/acp/server.rs` (which grows by
~3000 lines per the @@ markers, much of it the new ACP↔RMCP conversion
machinery and a substantial expansion of the chain-grouping flow inside
`tool_chain_meta` / its callers at lines 224-343). This is a lot of
change to land in one PR. As infrastructure work specifically labeled in
the PR description ("this makes the UI contract explicit before any
grouped-chain or tool-card redesign lands"), it's defensible, but the
review reviewability is bounded by the reviewer's tolerance for a
2k-line ACP-server diff.

**`extract_timeout_from_meta` signature change** (`server.rs:188-194`).
The function went from `Option<Meta>` to `Option<AcpMeta>` — a subtle
type-domain rename. If any downstream caller still passed the rmcp
`Meta` shape, it would fail to compile. The diff doesn't show every
call site, so worth confirming the rename was applied uniformly. Same
for `_get_allowed_mcp_servers_from_mcp_server_names` and the
`iter_known_server_prefixes` factoring.

**Replay-determinism contract.** The PR claims live and replay now
produce identical UI state, *but* the live path additionally has access
to streaming side-channels (e.g. token-stream context) that replay
doesn't. If any of those side-channels feed into the chain-summary
computation (e.g. "summary is derived from the most recent assistant
text chunk"), replay will silently disagree. The new tests
(`acpNotificationHandler.test.ts:78+, 379 new lines`) should be checked
for this — at minimum one test should assert "replay-then-live and
live-then-replay produce the same final message content tree."

**Provider field on `CodexProvider`** (`crates/goose/src/providers/codex.rs:60-160`).
The PR adds fields to a provider struct. Worth confirming this doesn't
break serialization for existing saved provider configs (i.e. that the
new field has a `#[serde(default)]` or is `Option<...>`); the diff
context is too narrow to tell, but the test additions at
`tests` (line 1023+) should cover round-trip.

**Default chain ID when none is supplied.** Live tool calls that aren't
explicitly part of a chain still need *some* chain id for the
metadata-keyed UI grouping to behave. The PR doesn't show the
chain-id assignment policy in the visible diff context; if it's
"unique-per-call when not supplied," that's fine; if it's "all
unkeyed calls share the empty string," that's a UI bug waiting to
happen.

## Verdict

**`needs-discussion`**

The directional design is right — make the contract explicit, share a
single set of builders between live and replay, derive titles/chain
metadata from authoritative ACP fields rather than per-path inference.
The conversion-helper machinery in `server.rs` is solid Rust. The
test coverage in `acpNotificationHandler.test.ts` is substantial.

But this PR is a load-bearing infrastructure change ("the UI contract
before any grouped-chain or tool-card redesign lands") that touches the
ACP server, both providers' input shapes, two UI handlers, and a new
metadata module — and the chain-grouping policy itself isn't visible
in the trimmed diff. Before merge I'd want:

1. **Explicit documentation of the chain-id assignment policy.** What
   makes two tool calls part of the same chain? Same parent message?
   Same agent turn? Provider-supplied? The metadata format is now
   load-bearing for grouped UX, so the policy needs to be in the PR
   description, ideally with a test case for each shape.
2. **A "live ↔ replay round-trip" test.** Take a recorded session,
   replay it into UI state, then re-emit the resulting notifications
   as if live, then replay again — assert the second replay produces
   the same UI state as the first. This is what the PR's correctness
   claim actually demands.
3. **Confirm `extract_timeout_from_meta`'s `Meta` → `AcpMeta` rename
   is exhaustive across callers.** Cargo will catch this at compile
   time, but worth a comment in the PR description acknowledging the
   type-domain change is intentional.
4. **`tracing::warn!` on the `Role::_` arm.** Silently collapsing
   unknown ACP roles to `Role::User` is the right behavior, but a
   warn-level log helps surface protocol drift early.

Once the chain-id policy is documented and the round-trip test lands,
this is a `merge-after-nits`. The conversion machinery itself is good
work and would be costly to re-do.

## What I learned

The "live and replay produce identical UI state" property is a useful
forcing function for any agent-UI design — if the replay path can't
reconstruct the live UI from persisted notifications, then either the
notifications are missing data (the failure mode this PR fixes) or the
live path is using out-of-band signals it shouldn't (the failure mode
the PR's tests need to verify isn't latent). The right shape, which
this PR moves toward, is: persist enough ACP metadata that replay is a
pure function of the notification stream, then make live use the same
deserialization path so any rendering bug shows up identically in both
modes. The hard part — and the open question on this PR — is the
chain-id assignment policy, because chain grouping is a UI feature
whose semantics depend on agreement between the producer (ACP server)
and consumer (UI), not just on schema correctness.
