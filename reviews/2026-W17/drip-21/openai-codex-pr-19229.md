# openai/codex#19229 — Add agent graph store interface

- **PR:** https://github.com/openai/codex/pull/19229
- **Head SHA:** `d5ec0216cf9c8696e9b42c4b922f68086789cfef`
- **Files:** 9 (+468/-0). New crate `codex-agent-graph-store`.
- **Verdict:** **merge-after-nits**

## Context

Sub-agent / multi-agent v2 in codex persists parent/child topology
(thread spawn edges) in the local SQLite-backed `StateRuntime`. As
the sub-agent feature grows toward a remote-orchestration story
(referenced obliquely by the "follow-up gRPC store" mentioned in
the PR body), continued direct coupling of orchestration code to
`StateRuntime`'s SQLite-specific helpers is the obvious next bottleneck.
This PR introduces the `AgentGraphStore` trait as the boundary, with
a `LocalAgentGraphStore` that wraps the existing `StateRuntime` so
no behavior changes today.

## Problem

Today, the call sites that `upsert_thread_spawn_edge`,
`set_thread_spawn_edge_status`, `list_thread_spawn_children_with_status`,
and `list_thread_spawn_descendants` reach into `StateRuntime`
directly. That means:

- The orchestration code can't be tested without a real `StateRuntime`
  (which means a real SQLite file on disk).
- A remote graph backend can't be substituted without rewriting
  every call site.
- The graph contract is defined implicitly by what `StateRuntime`
  happens to expose, not by an explicit narrow interface.

## Design — what's introduced

### Crate layout (visible in diff)

- `agent-graph-store/Cargo.toml` — new `codex-agent-graph-store`
  crate with `async-trait`, `codex-protocol`, `codex-state` as deps;
  `pretty_assertions`, `tempfile`, `tokio` as dev-deps.
- `src/lib.rs` — re-exports `AgentGraphStore`, `LocalAgentGraphStore`,
  `AgentGraphStoreError`, `AgentGraphStoreResult`,
  `ThreadSpawnEdgeStatus`.
- `src/error.rs` — two-variant error enum (`InvalidRequest`,
  `Internal`) carrying user-facing messages.
- `src/store.rs` — the trait surface (4 methods).
- `src/local.rs` — the SQLite-backed impl, ~340 lines including
  tests.
- `src/types.rs` — `ThreadSpawnEdgeStatus { Open, Closed }`.
- `Cargo.toml` workspace registration + `BUILD.bazel`.
- `Cargo.lock` regen + `cargo-shear` ignore-list entry (the new
  crate would otherwise be flagged unused until the migration PR
  lands).

### Trait surface (`store.rs` via re-export)

The four methods on `AgentGraphStore`:

```rust
async fn upsert_thread_spawn_edge(parent, child, status) -> Result<()>
async fn set_thread_spawn_edge_status(child, status) -> Result<()>
async fn list_thread_spawn_children(parent, status_filter: Option<…>) -> Result<Vec<ThreadId>>
async fn list_thread_spawn_descendants(root, status_filter: Option<…>) -> Result<Vec<ThreadId>>
```

This is a faithful narrowing of `StateRuntime`'s graph methods. The
`Option<ThreadSpawnEdgeStatus>` filter on the two list methods is
the right shape for a graph traversal API — the implementation
collapses both `Open` and `Closed` queries to two sub-calls when
the filter is `None`, then sorts.

### Local impl (`local.rs:35-110`)

The `LocalAgentGraphStore::list_thread_spawn_children` implementation
when `status_filter == None` is the most interesting code path:

```rust
let mut open_children = self
    .state_db
    .list_thread_spawn_children_with_status(parent_thread_id, Open)
    .await?;
let closed_children = self
    .state_db
    .list_thread_spawn_children_with_status(parent_thread_id, Closed)
    .await?;
open_children.extend(closed_children);
open_children.sort_by_key(std::string::ToString::to_string);
Ok(open_children)
```

Two sequential SQL round trips + a sort — simple, but worth a note
(below). For `list_thread_spawn_descendants` the underlying
`StateRuntime` already has a no-filter overload, so that branch is a
single call.

## Strengths

- **Crate boundary is the right shape.** Four methods, two error
  variants, one type. Nothing extraneous. This is exactly what a
  storage-neutral interface should look like at v0.
- **`LocalAgentGraphStore` has no orchestration logic in it.** It
  faithfully forwards to `StateRuntime`. That keeps the swap
  surface simple when the gRPC implementation lands.
- **Test fixture is well-isolated.** `state_runtime()` async helper
  in `local.rs:170-180` returns a `TestRuntime` holding both the
  `Arc<StateRuntime>` and the `_codex_home: TempDir` so the temp
  dir lives as long as the runtime — the standard pattern for
  avoiding "TempDir dropped before the SQLite file is closed"
  flakes.
- **`cargo-shear` ignore is honest** — the crate is unused at this
  PR but will be consumed in the migration PR. Better to declare
  this explicitly than to have CI break on the next sweep.

## Risks / nits

1. **`list_thread_spawn_children` with `None` filter does two
   queries + sort.** That's fine for `O(small)` topologies but if
   any caller starts using this on a wide root, the
   `Open + Closed + sort` approach becomes the obvious hotspot. A
   later optimization — push the filter into a single
   `list_thread_spawn_children_all` underlying call — is easy if
   the underlying state DB grows the method, but flag this as a
   known scaling boundary in a code comment so the next reader
   doesn't have to rederive it.
2. **`AgentGraphStoreError` has only `InvalidRequest` and
   `Internal` variants.** No transient/retryable distinction. For
   a future remote impl that's the wrong granularity — a
   `Transient { retryable: bool }` variant should be added before
   the gRPC store lands, otherwise every gRPC retry policy will
   either retry-internal-errors (wrong) or never-retry (also wrong).
   The fix is much cheaper now (one variant) than after the gRPC
   impl ships.
3. **`internal_error(impl Display) -> AgentGraphStoreError`** loses
   the underlying error chain — it stringifies. For a v0 trait this
   is acceptable, but I'd rather see `#[source]` chaining preserved
   so logs can show the SQLite error class:
   ```rust
   #[error("agent graph store internal error")]
   Internal(#[source] Box<dyn std::error::Error + Send + Sync>),
   ```
4. **Test for `list_thread_spawn_descendants` ordering** — the PR
   body claims "stable breadth-first descendant ordering" and the
   visible test list includes "stable breadth-first descendant
   ordering", but the test code itself isn't fully visible. If
   that ordering is supplied by `StateRuntime`, the trait contract
   should document that it inherits from the underlying impl —
   otherwise a future remote impl is free to return arbitrary
   orderings and silently break callers that rely on BFS.
5. **`set_thread_spawn_edge_status` takes only `child_thread_id`.**
   Reasonable, since the child uniquely identifies the edge in this
   model — but if the schema ever grows multi-parent edges, this
   signature is locked in by the trait. Worth a comment that the
   v0 model assumes a child has at most one parent.
6. The async-trait dependency is fine for now; if dispatch
   performance ever matters, GAT-based async-fn-in-trait could
   replace it. Not a blocker.

## Verdict — merge-after-nits

This is a good, small "carve out the seam" PR — the kind that costs
~500 lines now and saves several thousand in the migration PR that
follows. The two nits worth addressing before merge are the error
taxonomy (add a transient variant) and the BFS ordering contract
(document it on the trait, not just in the test name). Everything
else is acceptable as a v0 boundary.

## What I learned

Introducing a storage-neutral interface *before* the second backend
exists is the cheapest moment to do it: there's only one implementation
to migrate, no production callers depend on the trait shape yet, and
the test fixture can be a thin wrapper over the local impl. The
canonical mistake is to defer the trait until the gRPC backend forces
it — at that point every call site has to be rewritten under
schedule pressure with the new trait shape baked in. The "Open +
Closed + sort" naive impl when `status_filter = None` is also a
common shape: it's the right starting point because it preserves the
one-impl-many-readers property, but it's also exactly the shape that
needs a flag-or-comment so the next reader knows it's a known
scaling boundary, not a finished design.
