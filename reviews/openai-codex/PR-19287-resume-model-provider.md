# PR-19287 — Restore persisted model provider on thread resume

[openai/codex#19287](https://github.com/openai/codex/pull/19287)

## Context

Tiny patch in `codex-rs/app-server/src/codex_message_processor.rs`
inside `merge_persisted_resume_metadata`. Previously, on resuming a
saved thread, only `typesafe_overrides.model` was restored from the
persisted metadata; `model_provider` was left as whatever the new
session's defaults dictated. The one-line addition

```rust
typesafe_overrides.model_provider = Some(persisted_metadata.model_provider.clone());
```

threads the persisted provider back into the override block. Tests are
expanded across the existing 4–5 resume scenarios in `mod tests` to
assert `model_provider` matches expectation in each branch (mock
provider in the persisted-metadata case, `None` when there is no
persisted metadata).

## Why it matters

Without this, resuming a thread that was created against, say, an
Anthropic deployment could silently re-route subsequent turns through
the user's *current* default provider (OpenAI, Bedrock, whatever).
That breaks tool-call schemas, billing attribution, and reproducibility
of the conversation — the same model name (`gpt-5.1-codex-max`) can
exist behind multiple providers and have different behaviors.

## Strengths

- Bug is exactly one missing field assignment; the fix is exactly one
  line. Minimal blast radius.
- The test expansion is comprehensive: every existing test in the same
  module gained a `typesafe_overrides.model_provider` assertion (4
  branches: persisted-metadata-with-everything, persisted-with-only-
  model, persisted-with-nothing, persisted-with-only-provider). That
  pattern catches regressions where someone tightens the assignment
  back to a conditional.
- Uses `.clone()` on the `String` rather than mutating in place — keeps
  `persisted_metadata` immutable and reusable downstream.
- Symmetric with how `model` is already handled two lines above; the
  new code reads as "obvious follow-up that should have been there".

## Concerns / risks

- Always wrapping in `Some(...)` means a persisted thread will *always*
  override the runtime provider, even if the runtime user has
  intentionally switched providers (e.g. they exported credentials for
  a different vendor and want to migrate the conversation). There's no
  escape hatch in the diff. The `None` branches in the test only fire
  when `persisted_metadata` itself is absent — which is fine for now
  but worth flagging in the PR description so reviewers know the policy
  is "persisted wins, period".
- No test asserts the *interaction* with a runtime override that also
  sets `model_provider` — i.e. if the caller already populated
  `typesafe_overrides.model_provider = Some("anthropic")` before
  `merge_persisted_resume_metadata` runs, this line silently clobbers
  it. From the function name "merge", a caller might expect runtime
  overrides to win.
- Provider IDs are `String`s; if a persisted provider was renamed or
  removed in a later release (e.g. `azure_openai_v1` → `azure_v2`), the
  resume will fail at provider lookup with a less-helpful error than
  "provider not found, falling back to default".

## Suggested follow-ups

- Add a precedence test: pre-populated `typesafe_overrides.model_provider
  = Some("X")` + persisted `"Y"` — assert which one wins, and document
  it in the function doc-comment.
- Consider gating the assignment on `typesafe_overrides.model_provider.
  is_none()` if "runtime override wins on an explicit set" is the
  desired policy. Either way, write down the chosen policy.
- Add a graceful-degradation path: if the persisted `model_provider`
  doesn't exist at resume time, log a warning and fall through to the
  default provider rather than exploding mid-turn.
- Mirror this change for any other persisted-metadata fields that are
  currently *not* restored on resume (reasoning effort already is —
  worth grepping `persisted_metadata.` in this function for other gaps).
