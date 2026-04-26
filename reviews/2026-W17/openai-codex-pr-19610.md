# openai/codex #19610 — Support end_turn in response.completed

- **Repo**: openai/codex
- **PR**: #19610
- **Author**: andmis (Andrey Mishchenko)
- **Head SHA**: a21212b0dda597e01a417536dc23fd2d43124f17
- **Link**: https://github.com/openai/codex/pull/19610
- **Size**: ~190 diff lines across 6 files: `cli/src/responses_cmd.rs`,
  `codex-api/src/common.rs`, `codex-api/src/sse/responses.rs`,
  `codex-api/tests/sse_end_to_end.rs`, `core/src/client.rs`,
  `core/src/session/turn.rs`, plus a `Cargo.lock` adjustment.

## What it changes

Adds an optional `end_turn: Option<bool>` field to the SSE
`response.completed` event end-to-end through the codex pipeline.

1. **Wire decoding**: `ResponseCompleted` SSE struct
   (`codex-api/src/sse/responses.rs:124-127`) gets a new
   `#[serde(default)] end_turn: Option<bool>` field, which is
   threaded through `process_responses_event` (`:382-388`) into
   the `ResponseEvent::Completed` constructor.
2. **Public event type**: `ResponseEvent::Completed`
   (`codex-api/src/common.rs:81-87`) gains `end_turn:
   Option<bool>`. Comment: "Did the model affirmatively end its
   turn? Some providers do not set this, so we rely on fallback
   logic when this is `None`."
3. **Plumbing through core**: `client.rs:1655-1685` updates
   the destructure-and-rebuild loop to carry `end_turn` through
   the channel send, unchanged.
4. **The actual semantic use** is in
   `core/src/session/turn.rs:2132-2150`:
   ```rust
   if let Some(end_turn) = end_turn {
       needs_follow_up |= !end_turn;
   }
   ```
   I.e. if the provider set `end_turn=false`, force another
   sampling iteration; if `end_turn=true`, don't change
   `needs_follow_up`; if `None`, fall through to the existing
   inference logic unchanged.
5. **Test fixtures updated**: every existing test that
   destructures or constructs `Completed` (~7 sites) gets the
   new field added — most as `end_turn: None` (asserting backwards
   compat: the default-deserialise path produces `None`).

## Strengths

- **Backwards-compatibility shape is right.** `Option<bool>` +
  `#[serde(default)]` means any provider that doesn't emit
  `end_turn` continues to flow through the existing fallback
  logic. The `if let Some(end_turn)` guard at
  `turn.rs:2147-2149` is the surgical "only act on explicit
  signal" pattern — without it, a provider that emits
  `end_turn: false` to mean "I want to be inferenced again"
  would have been collapsed to "no signal" by an `Option ::
  unwrap_or(...)`-style coalesce.
- **`needs_follow_up |= !end_turn`** is the right composition
  with the existing inference-derived `needs_follow_up` —
  bitwise-OR means "either the existing logic OR the explicit
  signal can trigger a follow-up", but an explicit `end_turn:
  true` is *not* allowed to *override* a follow-up that
  inference logic already requested. That's defensive in the
  right direction: trust the model's stop signal, but don't let
  it suppress a tool-call cycle the agent already needs.
- **Comprehensive test fixture updates** — all 7+ sites that
  construct or destructure `Completed` are updated. The
  end-to-end SSE test at `codex-api/tests/sse_end_to_end.rs:158-167`
  asserts `end_turn.is_none()` for the default case, pinning
  the backcompat contract.
- **Per-event documentation** at `common.rs:84-86` calls out
  the contract: "Some providers do not set this, so we rely on
  fallback logic when this is `None`."

## Concerns / asks

- **`cli/src/responses_cmd.rs:81-82` ships a literal
  `FIXME(andrey): Make this actually work + test it, before
  merging.` comment** alongside the `end_turn: _` discard:
  ```rust
  codex_api::ResponseEvent::Completed {
      response_id,
      token_usage,
      // FIXME(andrey): Make this actually work + test it, before merging.
      end_turn: _,
  } => {
  ```
  This is the literal author-written marker that the PR is not
  ready. Whatever `responses_cmd` is supposed to surface
  (presumably `end_turn` in the JSON output for the `responses`
  CLI subcommand), it isn't surfaced and isn't tested. Cannot
  merge while the author has explicitly asked themselves not
  to.
- **No test for `end_turn=Some(false)` driving
  `needs_follow_up`.** The wire-decode and the channel-pass
  paths are tested (via the destructure asserts), but the
  actual semantic use at `turn.rs:2147-2149` — the only line
  that does work — has no test. A unit test injecting an SSE
  `Completed` with `end_turn: Some(false)` and asserting the
  outer turn loop fires a second sampling pass would close
  this.
- **No test for `end_turn=Some(true)` *not* suppressing an
  already-requested follow-up.** This is the subtle policy
  decision encoded by `|=` (vs. `=`). A test that sets up a
  pending tool-call cycle and asserts an explicit `end_turn:
  true` does NOT cancel it would lock the policy in.
- **Provider-side documentation is missing.** Which providers
  emit `end_turn`? OpenAI Responses spec doesn't appear to
  define this field on `response.completed` in the public docs
  (as of late W17 2026). If this is for a specific
  in-development provider, mention which — otherwise reviewers
  can't tell whether the wire field name is correct.
- **`codex-utils-absolute-path` removed from the `Cargo.lock`
  diff** (line 2873) but no production-code change explains why
  this dep was dropped. Either it's a coincidental cleanup
  that should be in a separate PR, or it's load-bearing on
  another change in this branch that hasn't been spelled out.
  Worth a one-line note in the body.
- **`needs_follow_up` is currently scoped narrowly** in the
  match arm at `turn.rs:2132-2153` — the `if let Some(...)`
  block extends only one statement (`needs_follow_up |=
  !end_turn`). Fine, but if more end_turn-driven policy
  decisions land later, consider extracting a helper
  `compute_needs_follow_up(needs_follow_up, end_turn)` so the
  policy lives in one place.

## Verdict

**request-changes** — the design is right (`Option<bool>` +
`|=` composition) and the test surface for the wire layer is
solid, but the **author has self-tagged `responses_cmd.rs:81`
with `FIXME(andrey): Make this actually work + test it,
before merging`**. That's an unambiguous "do not merge" from
the author. Resolve the FIXME (surface `end_turn` in the
`responses` CLI output and add a test), add a turn-loop
test for the semantic effect at `turn.rs:2147-2149`, and
this is mergeable.

## What I learned

When wiring an explicit-signal field through a deeply-layered
pipeline (SSE → event enum → channel → loop), the right shape
is `Option<T>` with `#[serde(default)]`, and the right
composition operator at the use site is one that *adds* to
existing inference (here, `|=`) rather than *overrides* it
(`=`). The composition choice is the most opinion-heavy line
in the whole PR and deserves a test in its own right. The
related lesson is that authors marking their own code with
literal `FIXME` markers should be auto-rejected by review
tooling — it's the strongest possible "not ready" signal and
it's right there in the diff.
