# Review: openai/codex PR #20719

- **Title:** Use responses request helpers for compact requests
- **Author:** aibrahim-oai (Ahmed Ibrahim)
- **Head SHA:** `0ae33ffe0b39b24f350fb98e33da300c7ca93ed1`
- **Verdict:** merge-as-is

## Summary

`/responses/compact` was building its own request payload and headers
separately from `/responses`. This PR routes compact through the same
`ResponsesApiRequest` builder + `build_responses_request_body` /
`build_responses_request_headers` helpers used by normal `/responses`,
then nulls out the small set of fields compact rejects (`store`,
`stream`, `include`, `client_metadata`). Removes the bespoke
`CompactionInput` payload type entirely. 807 added / 251 deleted
across 14 files; integration test snapshots cover varied turn history.

## Specific-line comments

- `codex-rs/codex-api/src/common.rs:23-37` (`CompactionInput`
  removed) — the bespoke compact-only struct goes away. This is the
  whole point: now there's exactly one wire-shape definition for the
  responses family.
- `codex-rs/codex-api/src/common.rs:159-170` — `store`, `stream`,
  `include` change from required (`bool`/`Vec<String>`) to
  `Option<...>` with `skip_serializing_if = "Option::is_none"`. This
  is the load-bearing change: compact sets these to `None` and they
  drop from the JSON; normal responses sets `Some(true)` /
  `Some(stream_value)` / `Some(includes)` and they serialize as
  before. Wire format is preserved on the normal path.
- `codex-rs/codex-api/src/common.rs:188` — `tool_choice` is removed
  from `ResponseCreateWsRequest` along with the line in the `From`
  impl. **This is a behavior change for the WS responses transport.**
  Verify a downstream consumer isn't reading `tool_choice` off the WS
  wire shape; if so this is a breaking change that needs a deprecation
  beat.
- `codex-rs/codex-api/src/endpoint/compact.rs:50-66`
  (`compact_request`) — the new high-level entry point reuses
  `ResponsesOptions` and the shared helpers. Old `compact_input`
  still exists (line 47 area) for backward compat. Worth marking
  `compact_input` `#[deprecated]` if there's a planned removal
  window.
- `codex-rs/codex-api/src/requests/responses.rs:+8-39`
  (`build_responses_request_body` / `build_responses_request_headers`)
  — the extraction is clean. `build_responses_request_body` checks
  `request.store == Some(true)` before calling `attach_item_ids` —
  correct, since compact doesn't store and therefore doesn't need
  Azure id rewriting.
- `codex-rs/core/tests/suite/compact_remote.rs` (+278) and the two
  `*_request_diff.snap` snapshots — this is the right test shape:
  drive the real helper path for 5 varied turns, capture the last
  normal `/responses` request and the following `/responses/compact`,
  and snapshot the full JSON body diff. Future request-shape changes
  on `/responses` will fail-loud in this snapshot if they don't also
  flow through to compact.

## Risks / nits

- The `tool_choice` removal from `ResponseCreateWsRequest` and the
  parent `ResponsesApiRequest` (line 162 in the diff) is the one
  thing worth callout. Compact rejected it; normal responses also
  doesn't appear to need it on the WS path. Confirm this isn't
  load-bearing for any provider variant before merging.
- No locally-run test results per PR body ("Not run locally per repo
  guidance; relying on GitHub CI"). The snapshot test design is good
  enough that this is acceptable, but a maintainer should verify CI
  green before merging.
- The shared body/header builders are now private (`pub(crate)`) to
  the codex-api crate. If `core` ever wants the same helpers for a
  third responses-shaped endpoint (e.g. a hypothetical
  `/responses/score`), they'll need to be re-exported. Fine for now.

## Verdict justification

Clean, well-motivated dedup. Snapshot-test design specifically locks
in the parity property the PR claims. The `tool_choice` removal is
worth one final check, but if it's truly unused on the WS path this
is a straight-up improvement with no downside. **merge-as-is** —
landed against a reasonable maintainer review of the CI snapshot
output.
