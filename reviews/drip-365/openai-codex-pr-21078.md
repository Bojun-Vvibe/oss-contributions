# openai/codex #21078 — [codex] Add loaded thread summaries for TUI backfill

- **Head SHA:** `1408437644a16825f2a3ace9ea94f8c09006a7c1`
- **Base:** `main`
- **Author:** canvrno-oai
- **Size:** +402 / −138 across 18 files (5 generated schema, 1 protocol, 2 server, 2 server tests, 4 TUI, 4 TS schema)
- **Verdict:** `merge-after-nits`

## Summary

Adds an opt-in `includeSummaries: bool` parameter to the `thread/loaded/list`
v2 RPC. When set, the response includes a parallel `summaries: Vec<ThreadLoadedSummary>`
array with `{id, parentThreadId, agentNickname, agentRole}` for each returned
thread id. The TUI's loaded-subagent backfill drops 98 lines of per-thread
`thread/read` request fanout (`codex-rs/tui/src/app/loaded_threads.rs`) in
favor of walking the summary parent edges from a single batched call.

## What's right

- **Single-source backend ownership.** `ThreadLoadedSummary` is built from the
  live thread config snapshot inside the app-server's
  `thread_processor.rs` (the request returns 49 net new lines — a
  `match params.include_summaries { true => build, false => Vec::new() }`
  fork that preserves the existing `data: Vec<String>` shape exactly). The
  client never has to assemble parentage by side-channel.

- **Backwards-compatible serde contract.** At `protocol/v2.rs:4396-4397` the
  new field is `#[serde(default, skip_serializing_if = "std::ops::Not::not")]`
  — old clients that omit `includeSummaries` get `false` server-side and the
  server omits the field on the wire when false. At
  `protocol/v2.rs:4408-4410` the response field is `#[serde(default)]` so a
  pre-summary client deserializing a (hypothetical) summary-bearing response
  defaults to an empty Vec instead of erroring. Both directions are forward
  and backward compatible — clean wire-protocol evolution.

- **Schema regen is consistent.** All four generated schema artifacts — root
  `ClientRequest.json`, two `codex_app_server_protocol.*.schemas.json` files,
  and per-message `v2/ThreadLoadedListParams.json` /
  `v2/ThreadLoadedListResponse.json` — show the same `includeSummaries` /
  `summaries` / `ThreadLoadedSummary` additions. The new TS export at
  `schema/typescript/v2/ThreadLoadedSummary.ts:5-21` matches the Rust struct
  field-for-field (`id: string`, `parentThreadId: string | null`, etc.) with
  the `// GENERATED CODE!` header preserved, and `schema/typescript/v2/index.ts`
  re-exports it alongside the existing types.

- **Doc + example updated together.** `app-server/README.md:150` documents the
  new opt-in, and the example block at `README.md:347-360` shows both a
  parent-less root thread and a child thread with `parentThreadId`,
  `agentNickname`, `agentRole` set — matching exactly the shape an
  `AgentControl`-spawned subagent would emit.

- **TUI side is a real simplification.** `loaded_threads.rs` shrinks 59/+98−,
  net −39 lines, and `session_lifecycle.rs` only grows by 33 net. The
  removed code was the per-thread `thread/read` loop the PR description calls
  out — that was the actual perf hazard during heavy multi-subagent sessions
  where loaded count is in the dozens.

## Concerns / nits

1. **`summaries` and `data` are independent vectors with no documented length
   invariant.** When `includeSummaries: true`, every consumer expects
   `summaries.len() == data.len()` and `summaries[i].id == data[i]`. The
   protocol struct at `protocol/v2.rs:4404-4411` declares both as plain
   `Vec`s with no comment pinning the invariant, the JSON schema description
   at `v2/ThreadLoadedListResponse.json:51-54` says only "Loaded-thread
   summaries for the returned page when `includeSummaries` is true" — it
   doesn't promise order alignment with `data`. Two tighter alternatives
   either (a) document the invariant explicitly in both the doc-comment and
   the JSON schema description, or (b) restructure as
   `Vec<ThreadLoadedEntry { id, summary: Option<…> }>` so the alignment is
   structural not by-convention. (a) is enough; preserve `data` for
   backwards-compatibility.

2. **No explicit handling of `data` ↔ `summaries` mismatch on the server
   build path.** The PR adds `thread_processor.rs:1896+ (49 net lines)` to
   build summaries from live thread configs. If the loaded-thread set
   mutates between building `data` (the page) and building `summaries`
   (under whatever lock is held inside the processor), the response can
   ship a `summaries` entry without a matching `data` entry or vice versa.
   Worth a code comment confirming both lists are built from the same
   snapshot under the same read lock — or, if not, an explicit
   `summaries.iter().all(|s| data.contains(&s.id))` debug_assert before
   send.

3. **`agentNickname` and `agentRole` are both `Option<String>` and both
   "AgentControl-spawned subagent" attributes.** A correctness invariant
   not in the schema: if `parentThreadId.is_none()` (root thread), then
   `agentNickname.is_none()` and `agentRole.is_none()`. Doc-comment on
   `ThreadLoadedSummary` at `protocol/v2.rs:4421-4429` should state this so
   future TUI/client code doesn't write `if summary.agent_role.is_none()
   { … }` branches expecting subagent-vs-root semantics that the schema
   doesn't actually guarantee.

4. **`thread_loaded_list` test coverage is thin (+23/−3).** The PR-touched
   test file `app-server/tests/suite/v2/thread_loaded_list.rs` adds a
   single test for the `includeSummaries: true` path. Missing assertions:
   (a) that `includeSummaries: false` (or omitted) returns
   `summaries: Vec::new()`, (b) that the `summaries` order aligns with
   `data` order under pagination (cursor reuse), (c) that an unloaded
   thread that gets evicted between `data` build and `summaries` build is
   handled (per concern 2). The existing test likely covers (a) implicitly
   via default but it's worth an explicit `assert!(resp.summaries.is_empty())`
   regression.

5. **`schema/typescript/v2/ThreadLoadedListParams.ts:13`** — the TS file
   declares `includeSummaries?: boolean` (optional) but the underlying Rust
   uses `#[serde(default)]` which means it's `bool` not `Option<bool>` on
   the Rust side. Generators can flatten this safely either way; just
   confirm the TS-side optional-ness matches the OpenAPI generation
   convention used by the existing `limit`/`cursor` fields (which the diff
   shows as `cursor?: string | null` and `limit?: number | null`). If the
   generator chose `?` for `Option<u32>` and `?` for `bool` with
   `serde(default)`, fine; if not, the asymmetry is worth a one-line
   comment in the protocol struct.

## Verdict rationale

Schema-evolution hygiene is good (concern 1 doc-only fix), TUI simplification
is real, and the v2 protocol change is opt-in by default. Concerns 2 and 4
(snapshot-alignment + test coverage) are the only ones that touch correctness
and both are addressable without rework.
