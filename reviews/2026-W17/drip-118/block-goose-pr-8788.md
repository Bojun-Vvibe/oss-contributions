# PR #8788 — fix(acp): coalesce streaming chunks under one message id

- **Repo**: block/goose
- **PR**: #8788
- **Head SHA**: `02cc44e4`
- **Author**: smorina
- **Size**: +30 / -3 across 1 file
- **Verdict**: **merge-as-is**

## Summary

Fixes #8748 — every ACP-based provider (`claude-acp`, `amp-acp`,
`codex-acp`, `pi-acp`, and the github coding-assistant ACP)
renders streamed responses as many separate message bubbles in
the desktop UI rather than a single progressively-filling
assistant message. Root cause: `AcpProvider::stream` was yielding
each text chunk as a fresh `Message::assistant()`, which builds
a `Message` with `id: None` and a fresh `Utc::now()` timestamp
per yield. The desktop coalescing check in
`ui/desktop/src/hooks/useChatStream.ts` requires both
`lastMsg.id` truthy AND matching the incoming id, so with
`id: None` serialized as null/absent the coalesce branch is
never taken and every chunk renders as its own bubble. CLI
output is unaffected because the CLI doesn't depend on
matching message ids.

## Specific changes

- `crates/goose/src/acp/provider.rs:339-344` — new `fresh_text_run() -> (String, i64)` helper that returns `(uuid::Uuid::new_v4().to_string(), chrono::Utc::now().timestamp())`. Free function, not a method — appropriate for what's effectively a tagged-tuple constructor.
- `crates/goose/src/acp/provider.rs:433-440` — declares `let mut text_run: Option<(String, i64)> = None;` at the top of the `try_stream!` block with a paragraph-long comment explaining the contract: "Contiguous Text/Thought chunks share one id + timestamp so Desktop's useChatStream coalesces them into a single bubble. Reset on any tool-call or permission event: post-tool text must get a fresh timestamp, because SessionManager reloads messages ordered by `created_timestamp`. Reusing the stream-start timestamp would make post-tool text reload before the tool call." This is the *right* shape for a comment — names the invariant, names the consumer that depends on it (`SessionManager` ordered by `created_timestamp`), and explains the failure mode if the invariant is violated.
- `crates/goose/src/acp/provider.rs:445-462` — both `AcpUpdate::Text` and `AcpUpdate::Thought` now use `text_run.get_or_insert_with(fresh_text_run).clone()` to lazily allocate the run on first chunk, then build the message via `Message::new(Role::Assistant, ts, vec![]).with_text(...).with_id(id)` instead of `Message::assistant().with_text(...)`. The `get_or_insert_with` is the right primitive for "create if missing, return either way" without a separate `if let None` branch.
- `crates/goose/src/acp/provider.rs:467,495,525` — `text_run = None` is set at the entry to every other `AcpUpdate` variant arm: `ToolCallStart`, `ToolCallComplete` (under both the `rejected_tool_calls.remove(&id)` branch and the normal-completion branch), and `PermissionRequest`. The reset is what enforces the "post-tool text gets a fresh timestamp" invariant from the comment.

## Risks

- **`SessionManager` ordering invariant is now load-bearing on this code**: the comment correctly identifies that `SessionManager` loads messages ordered by `created_timestamp`, which is why post-tool text must get a new timestamp (otherwise the post-tool text reloads *before* the tool call that produced it). That invariant lives in `session_manager.rs` and isn't enforced by a type — if someone changes the ordering to id-based or insertion-order, the timestamp reset becomes either unnecessary or wrong. Not a blocker — the comment is the contract — but a regression test that reloads a session containing `[stream-text, tool-call, post-tool-text]` and asserts the order survives would pin both directions.
- **Contiguous Thought chunks share an id with adjacent Text chunks**: the diff has both `AcpUpdate::Text` and `AcpUpdate::Thought` sharing the same `text_run`. If the desktop UI ever decides to render Thought differently from Text (e.g. collapsed-by-default, separate panel), the shared id will surface them as the same coalesced bubble even though they're semantically different. The PR description doesn't address this — worth confirming with the desktop side that "Thought + Text in one bubble" is the intended UX. If not, a separate `let mut thought_run: Option<...> = None` would split the streams while keeping the within-stream coalesce.
- **`ToolCallComplete` resets the run regardless of whether the tool actually completed visibly**: under `rejected_tool_calls.remove(&id)` the diff still resets `text_run = None`. That's probably correct (a rejected tool still represents a logical break in the message stream), but a one-line comment naming that intent would prevent a future refactor from "optimizing" the reset out of the rejected-tool branch.
- **No new test in the diff**: the PR description says `cargo test -p goose --lib acp::` passes 100/100 plus manual verification in the desktop with `claude-acp`. For a 30-line fix to a streaming protocol that's load-bearing on UI rendering, an automated test that drives the stream and asserts message ids cluster as `[id1, id1, id1, tool-id, id2, id2]` would catch a coalesce regression before it ships. Cheap to add; cheap to skip if the maintainer team is comfortable with the manual signal.
- **`uuid::Uuid::new_v4()` is fine but unauditable**: each stream allocates a fresh random UUID for the text run. That's the right choice (no collisions, no global state), but the trace logs / debugging flow won't be able to correlate "this chunk belongs to that run" by inspection. Probably noise, not signal.

## Verdict

`merge-as-is` — the bug is well-localized, the fix is the
minimum-surface change, the comment names the invariant
explicitly (which is the gold standard for "code that looks
unnecessary but isn't"), and the `claude_code.rs` precedent
cited in the PR description is exactly the right "we already
do this, here's the pattern" framing for a reviewer to lean on.
The desktop-side coalescing depends on matching ids and matching
timestamps; this PR provides both, in the right shape, with the
right reset boundaries.

## What I learned

The pattern "shared identity across a contiguous run of related
events, reset on a logical boundary" shows up in every
streaming UI: speech transcription paragraphs, real-time
collaboration cursor strokes, log line groupings, and now ACP
text/thought chunks. The trap that tripped this diff's
predecessor is that `Message::assistant()` looks like a
constructor for "a single message" but is actually called
many times per logical message — every call mints fresh
identity, which is invisible at the call site. The right
defense at the type level would be a `MessageBuilder` that
takes `&mut self` and only mints identity on `.build()`, so
the code that streams chunks can't accidentally mint identity
per chunk. This PR doesn't go that far (and shouldn't, for a
bug fix), but it's the durable-shape question for whoever
owns the `Message` API.
