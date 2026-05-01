# openai/codex#20532 — feat(app-server): API proposal for better thread loading performance

- **PR**: https://github.com/openai/codex/pull/20532
- **Head SHA**: `0aec5b61efc72d16c8c310208f4e3b696fb0a8a7`
- **Size**: +292 / -12, 7 files (protocol crate + app-server consumer + README)
- **Verdict**: **needs-discussion**

## Context

API-proposal PR adding two new app-server endpoints (`thread/items/list`, `thread/item/content/read`) plus a `largeContent: "inline" | "deferred"` mode parameter on the existing `thread/read`, `thread/resume`, `thread/fork`, `thread/turns/list` calls. Goal: let clients page large thread histories without ever materializing every generated-image base64 blob in one response. The proposal is documented but the consumer surface (`bespoke_event_handling.rs`, `codex_message_processor.rs`, `in_process.rs`) shows only +30 lines of glue — implementation is mostly *not* present in this PR.

## What's right

- **`TurnItemsView` enum at `app-server-protocol/src/protocol/v2.rs:5282-5294`** — three explicit variants (`NotLoaded`, `Summary`, `Full`) replace the prior implicit "items is empty unless this came from resume/fork" comment. Every Turn carries its own loaded-state declaration so a client can distinguish "this turn has no items" from "this turn's items weren't loaded into this response." The exhaustive flip of every test-arm at `thread_history.rs:1170-2980` (every `Turn` literal now carries `items_view: TurnItemsView::Full`) confirms the enum is non-optional and forces the producer-side discipline at compile time.
- **`ImageGenerationContent` tagged union at `v2.rs:6107-6132`** — three-variant tagged union (`Inline { data_base64, mime_type, byte_length }`, `Deferred { content_id, mime_type, byte_length, width, height }`, `Unavailable { reason }`) replaces the prior "result is base64, trust me" string. The `Unavailable` variant is the right call for in-flight image generation (`thread_history.rs:583-587` uses `Unavailable { reason: "image generation has not completed" }` on the pending arm). New clients render from `content`; legacy `result: String` is preserved for compat with the policy "deferred mode expects this to be empty."
- **`LargeContentMode` at `v2.rs:4555-4565`** — opt-in, defaults to `Inline` for compat. Documented at four call sites (`ThreadResumeParams`, `ThreadForkParams`, `ThreadReadParams`, `ThreadTurnsListParams`) with consistent doc-comment template.
- **Summary-Turn contract in `thread/turns/list`** — README at `app-server/README.md:152` and the `ThreadTurnsListResponse` doc-comment at `v2.rs:4476-4480` both pin the same contract: each summary Turn includes only the first user message and final assistant message in `items` when available. Single source of truth; the next PR in the stack can implement against it without ambiguity.

## Risks / nits

- **Implementation surface is thin.** This PR adds the protocol (`+213` in protocol crate) but the consumer-side glue at `bespoke_event_handling.rs:+3`, `codex_message_processor.rs:+25`, `in_process.rs:+2` is barely-stub. The actual `thread/items/list` handler that pages items from rollout, the `content_id` allocation/lookup table, the `Deferred → byte_length/width/height` extraction, and the `Summary` Turn assembly logic are all absent. PR-as-merged would let a client request `thread/items/list` and receive an unimplemented-method error.
- **`ImageGenerationContent::Inline` duplicates `result: String`.** Every producer site at `thread_history.rs:585, 599, 1475, 6593` writes the same base64 to both `content: Inline { data_base64: image.result.clone() }` and `result: image.result.clone()`. Doubles memory + JSON size for inline mode (the *common* case during the compat window). The PR body notes "in deferred mode, expect `result` to be empty" but doesn't pin the producer-side discipline that *enforces* that in deferred mode — easy to drift.
- **`content_id` opacity is undefined.** `ThreadItemContentReadParams` carries `(thread_id, turn_id, item_id, content_id)` but the doc-comment only says "Opaque content identifier returned from a deferred item content placeholder." No spec on lifetime (does `content_id` survive across app-server restarts? across thread archive/unarchive? is it scoped per-process?), no spec on cardinality (one `content_id` per item, or per content fragment within an item?), no spec on auth (any client that has the `thread_id`+`turn_id`+`item_id` triple can request bytes — is that fine?). These are the questions a follow-up implementer will block on.
- **`backwardsCursor` documented identically for `thread/turns/list` and `thread/items/list`.** Two responses, same paragraph. Worth factoring the cursor-direction-flip semantics into a shared doc helper so they can't drift.
- **No JSON-schema regression test in this PR.** Adding `ThreadItemsList`, `ThreadItemContentRead`, `LargeContentMode`, `TurnItemsView`, and three `ImageGenerationContent` variants to the protocol surface should bump the snapshot of the shared `_lazy_openapi_snapshot` or similar. Diff doesn't show it.

## Verdict

**needs-discussion.** Protocol shape is clean and the `TurnItemsView` + `ImageGenerationContent` + `LargeContentMode` triple is well-factored, but this is API-proposal-only with no implementation behind it. Either (a) split into "protocol-only" + "implementation" PRs and merge the protocol with an explicit "endpoints return UnimplementedMethod" note in the README, or (b) hold this PR until the consumer-side handlers land. Also block on resolving the `content_id` lifetime + auth + scoping spec gaps before any client code starts depending on the surface.
