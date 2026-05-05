# openai/codex#21110 — [codex] add deferred image content apis

- Head SHA: `329222a4a73a60fee9560b46394c6cd8787214a5`
- Author: rhan-oai
- Verdict: **needs-discussion**

## Summary

Three coupled changes:

1. New image-generation large-content model with inline and deferred
   variants, while preserving the legacy `result` field for back-compat.
2. New experimental `thread/turns/items/list` and
   `thread/item/content/read` request/response types so clients can
   page through history without inlining multi-MB base64 image
   payloads.
3. Lets `thread/read` (history) responses return image metadata with
   `largeContent: "deferred"` instead of inline bytes.

The diff is large (~3.6k lines) but most of that is regenerated
schema JSON across the v1 and v2 protocol surfaces.

## Specific references

- `codex-rs/app-server-protocol/src/protocol/v2.rs` — new
  `ImageGenerationContent` and `LargeContentMode` types.
- `codex-rs/app-server-protocol/src/protocol/thread_history.rs` — new
  request shapes for `thread/turns/items/list` and
  `thread/item/content/read`.
- `codex-rs/app-server/src/request_processors/thread_processor.rs`
  and `thread_processor_tests.rs` — handler wiring + tests for the
  new endpoints.
- Generated JSON schemas across
  `codex-rs/app-server-protocol/schema/json/v2/Thread*.json`,
  `Turn*.json`, `Review*.json`, `ItemStartedNotification.json`,
  `ItemCompletedNotification.json` all gain the deferred-image
  variant in their `union`/`oneOf`.
- `codex-rs/app-server-protocol/schema/typescript/v2/LargeContentMode.ts`
  enumerates the new client-facing mode.

## Reasoning

The motivation — keep multi-MB base64 image payloads out of every
history response — is real and important for memory-constrained
clients (TUI, mobile/web ACP wrappers). The two new APIs are the
right shape: a list call that returns lightweight metadata + an
`itemId` and a separate fetch call that streams the bytes.

What pushes this into needs-discussion territory rather than a
straight nit-set:

1. **Cross-cutting protocol surface change.** The `largeContent:
   "deferred"` discriminant is now a possible value in many existing
   response types (`ThreadReadResponse`, `ThreadResumeResponse`,
   `ThreadForkResponse`, `ThreadStartResponse`, `TurnStartResponse`,
   `ReviewStartResponse`, `ItemStartedNotification`,
   `ItemCompletedNotification`, `ThreadListResponse`, ...). Every
   client that pattern-matches over the content variant needs to
   handle `deferred` or it'll silently render empty image cells. The
   PR description doesn't list which clients have been updated; this
   should be enumerated and gated, or rolled out behind a server
   capability flag so old clients don't see `deferred` until they
   advertise support.

2. **Fetch authorization and lifetime.** `thread/item/content/read`
   takes an `itemId` and returns bytes. What happens if the calling
   session doesn't own that thread? What happens after the thread is
   archived/deleted — is the `itemId` still resolvable, and for how
   long? The diff doesn't show explicit auth checks at the handler
   site (or I missed them in the noise); confirm before shipping.

3. **Caching/dedup story.** Multiple turns that reference the same
   generated image will all have distinct `itemId`s. There's no
   content hash exposed in the metadata, so a client can't cache by
   content. Probably fine for v1 but worth noting.

4. **`largeContent` is overloaded.** The same field name is now used
   on both the request side ("how should the server format large
   content") and the response side ("how was this content actually
   delivered"). Consider distinguishing — `largeContentMode` for the
   request, `deliveryMode` for the response — or document the dual
   role explicitly.

Once those are resolved (especially #1 and #2), this is a clear win.
