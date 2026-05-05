# openai/codex #21225 — app-server: ignore persist_extended_history param

- URL: https://github.com/openai/codex/pull/21225
- Head SHA: `ae63488d861be5da848135981b087e25f0965312` (`ae63488`)
- Author: @owenlin0 (Owen Lin)
- Diff: +84 / −22 across 4 files
- Verdict: **merge-after-nits**

## What this changes (and why this is the right migration shape)

Three protocol structs (`ThreadStartParams`, `ThreadResumeParams`,
`ThreadForkParams` in `app-server-protocol/src/protocol/v2.rs`) keep
their `persist_extended_history: bool` field with the
`#[experimental(...)]` and `#[serde(default)]` attributes intact, but
the doc-comment is rewritten from the previous
"If true, persist additional EventMsg variants to the rollout file"
language to "Deprecated and ignored by app-server. Kept only so older
clients can continue sending the field while rollout persistence
always uses the limited history policy." That preserves wire-format
compatibility — older clients sending the field continue to
deserialize successfully — while making the contract documentation
match the new semantics.

The behavioral pivot is at
`thread_processor.rs:993` (`persist_extended_history: false` is
hardcoded into the core `Op::CreateThread` payload regardless of what
the client sent) plus the matching `tracing` field at line 1005 also
pinned to `false`. The `submit_core_op` signature loses the
`persist_extended_history: bool` parameter entirely (line 877 in the
new file), removing it from the function's public surface within
app-server so future contributors can't plumb the legacy value back
through.

The `collect_resume_override_mismatches` helper at line 121-128 in
the pre-patch file used to flag a `persistExtendedHistory override
was provided and ignored while running` mismatch when a `thread/resume`
request tried to change the persistence mode mid-thread; that branch
is now deleted because there's no longer a meaningful difference to
diff against — the field is universally ignored, so reporting it as
an "override mismatch" would be both noisy and misleading.

## The deprecation-notice signal is the load-bearing UX piece

The new `send_persist_extended_history_deprecation_notice` helper at
`thread_processor.rs:842-852` fires a connection-scoped
`ServerNotification::DeprecationNotice` (with summary
`"persistExtendedHistory is deprecated and ignored"` and detail
`"Remove this parameter. App-server always uses limited history
persistence."`) only when the client explicitly sets the field to
`true` (`if persist_extended_history` gate at line 751 in the
post-patch view). This is exactly the right gate: clients sending the
field as `false` (the default) get no spam, clients still sending
`true` because they remember the old semantics get a one-shot
notification per thread/start so they know to remove it — and the
notice is sent BEFORE `submit_core_op` so the client can correlate it
with the same request.

The new `thread_start_deprecates_persist_extended_history_true` test
in `app-server/tests/suite/v2/thread_start.rs` (+41 lines)
specifically pins the notify-on-true behavior so future refactors
can't silently drop it.

## Concerns

(a) The `DeprecationNotice` is only emitted from `thread/start` per the
PR diff visible — the resume/fork paths also accept the field but the
diff snippet I saw doesn't show whether they fire the same notice on
explicit `true` (line 2214 of the post-patch file is referenced but
truncated in my fetch). Reviewer should confirm symmetric coverage
across all three entry points; if `thread/resume` and `thread/fork`
silently swallow `persist_extended_history: true` without notifying,
clients have a partial deprecation signal which is worse than no
signal because it teaches users that "removing the field is fine on
start but maybe required for resume" — an inconsistent mental model.

(b) The PR keeps the `#[experimental("thread/start.persistFullHistory")]`
attribute on the field. `experimental` and `deprecated` are different
lifecycle states; if there's an established `#[deprecated(...)]`
attribute pattern elsewhere in `app-server-protocol` it should be
applied here so generated client SDKs (TS, JSON Schema) surface the
deprecation in their own type systems, not just in the doc-comment
prose.

(c) The hardcoded
`persist_extended_history: false` at `thread_processor.rs:993` and
the matching `thread_start.persist_extended_history = false` tracing
attribute at `:1005` mean the OTel signal that previously distinguished
"client requested extended history" from "client did not" is now
pinned to the constant `false`. That telemetry is permanently dead;
if there's value in counting how many clients are still sending
`true` (to know when it's safe to actually remove the field from the
protocol), capture that as a separate metric on the deprecation-notice
emission path before this lands.

The migration is well-shaped and the test coverage is right.
Once the resume/fork notice symmetry is confirmed (or fixed), this
ships.
