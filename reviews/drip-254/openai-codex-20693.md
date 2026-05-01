# openai/codex #20693 — [codex] Preserve image detail in app-server inputs

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20693
- **HEAD SHA:** `5fad2945b670c00faf5b13f055d7a0269890711e`
- **Author:** fjord-oai
- **Verdict:** `merge-after-nits`

## What the diff does

Plumbs an optional `ImageDetail` (`auto` / `low` / `high` / `original`)
through the app-server `UserInput::{Image, LocalImage}` variants into
core's `UserInput`, then through `From<Vec<UserInput>> for ResponseInputItem`
so that the chosen detail reaches the OpenAI Responses API
`InputImage { detail }` content item, with `Original` carrying the
extra semantic of "skip the resize step that would otherwise downscale
to fit the prompt window."

Files of note:

- `protocol/src/user_input.rs:23-40` — the canonical `UserInput`
  enum gains `detail: Option<ImageDetail>` on both `Image` and
  `LocalImage`, with `#[serde(default, skip_serializing_if =
  "Option::is_none")]` and `#[ts(optional)]` so on-wire and TS
  clients see a strict-additive change (existing payloads without
  `detail` keep deserializing fine, defaulting to `None`).
- `protocol/src/models.rs:1063-1117` — splits the existing
  `local_image_content_items_with_label_number` into a thin
  back-compat wrapper that calls a new
  `_with_label_number_and_detail` helper, where the line that used
  to be hardcoded `detail: Some(DEFAULT_IMAGE_DETAIL)` at `:1080`
  now takes the caller-supplied `detail`. The new
  `prompt_image_mode_for_detail` (`:1106-1112`) is the
  load-bearing branch: `ImageDetail::Original →
  PromptImageMode::Original` (skip resize), every other variant
  (`Auto`/`Low`/`High`) → `PromptImageMode::ResizeToFit`. Without
  this branch, "original" detail would still get downscaled before
  base64 encoding and the API would receive a smaller image than
  the operator asked for.
- `protocol/src/models.rs:1230-1265` — the
  `From<Vec<UserInput>> for ResponseInputItem` arms unpack the new
  field and `unwrap_or(DEFAULT_IMAGE_DETAIL)` so the legacy
  default-detail path is preserved when caller passes `None`.
- `app-server-protocol/src/protocol/v2.rs:5791-5860` — the v2
  envelope adds the same field (placed *before* `url`/`path` in
  the struct so the JSON-Schema property order is consistent
  across nested envelopes), and the `Into<CoreUserInput>` /
  `From<CoreUserInput>` arms forward `detail` lossless in both
  directions. Roundtrip locking test at `:10238-10299` constructs
  a `CoreUserInput::{Image,LocalImage} { detail:
  Some(ImageDetail::Original) }` and asserts the round-trip
  through v2 preserves it.
- `protocol/src/models.rs:2602-2627` —
  `image_user_input_preserves_requested_detail` is the dispositive
  unit test: data-URL `Image { detail: Some(Original) }` must
  produce `ContentItem::InputImage { detail: Some(Original) }` at
  index `1` of the assembled message content.
  `local_image_user_input_preserves_requested_detail` (`:2856-2891`)
  does the same for `LocalImage` *and* asserts the
  `prompt_image_mode_for_detail` mapping directly so the resize-vs-
  original branch is locked in.
- 22 schema-fixture JSON files regenerated under
  `app-server-protocol/schema/json/` — pure regen output from
  `cargo run -p codex-app-server-protocol --bin
  write_schema_fixtures`, all adding the `detail` anyOf-with-null
  property and the `ImageDetail` enum definition.
- ~12 call sites updated to pass `detail: None` (TUI tests,
  `chatwidget.rs`, `event_mapping.rs`, `core/tests/suite/*`,
  `analytics_client_tests.rs`, etc.) — purely mechanical
  field-addition fan-out.

## Why it's right

The plumbing shape is correct: the detail is set at the user-facing
typed gate (the `UserInput` enum) and forwarded losslessly through
every intermediate representation (v2 envelope → core → Responses
content item). `Option<ImageDetail>` rather than a defaulted
non-Option means existing clients that don't know about the field
get exactly the prior behavior (`unwrap_or(DEFAULT_IMAGE_DETAIL)`),
and adding the field as `#[serde(default, skip_serializing_if =
"Option::is_none")]` keeps the on-wire format strictly additive —
old serialized payloads deserialize unchanged, and new payloads
that don't set `detail` produce the same JSON they did before.

The `prompt_image_mode_for_detail` branch is the load-bearing piece
of new behavior. Without it, requesting `detail: Some(Original)`
would still emit `PromptImageMode::ResizeToFit`, the bytes would
be downscaled to the prompt window's resize threshold *before* the
base64 encoding, and the Responses API would receive an image
smaller than what the operator asked for — the `detail: Original`
on the wire would be a lie. The branch also correctly defaults
*every other* `ImageDetail` variant (including the explicit `auto`,
`low`, `high`) to `ResizeToFit`, which matches the OpenAI API
contract that `original` is the only fidelity tier that should
bypass downscaling.

The roundtrip test at `v2.rs:10238-10299` is the right test surface
for an envelope-conversion change: it constructs the core type with
a non-default `detail`, converts to v2, asserts the v2
representation has the field, then converts back and asserts the
core type matches. Two independent assertions (v2 and core) lock
both directions of the conversion.

The schema-fixture regen is conservative and consistent — every
v2 envelope that carries `UserInput` (`TurnStartParams`,
`ItemStartedNotification`, `ThreadReadResponse`, etc.) gets the
`ImageDetail` enum + `detail` anyOf-null property added in lockstep,
so an external TS/JS client regenerating from the fixtures sees a
consistent surface across all 16 message types.

## Nits

1. **`prompt_image_mode_for_detail` collapses three distinct
   detail tiers (`Auto`, `Low`, `High`) onto the same
   `ResizeToFit` mode.** That's correct *today* because resize
   policy is fidelity-agnostic at this layer, but the function
   name reads as "compute mode from detail" and a reader will
   reasonably expect that `Low` and `High` produce different
   modes. A doc comment clarifying "resize is only bypassed for
   `Original`; the `Auto`/`Low`/`High` distinction is forwarded
   verbatim to the API but doesn't affect local resizing" would
   prevent a future "shouldn't `Low` use a smaller resize
   threshold?" refactor that breaks the API-fidelity contract.

2. **No test for `Image { detail: Some(Auto/Low/High) }`** — the
   test pair (`image_user_input_preserves_requested_detail` and
   `local_image_user_input_preserves_requested_detail`) only
   covers `Some(Original)` and the `None` (default) cases. If a
   future refactor accidentally collapses `Auto`/`Low`/`High` to
   `Original` (or vice-versa), the existing tests don't catch it.
   Parametrize over all four `ImageDetail` variants for both
   `Image` and `LocalImage` so each tier gets locked.

3. **`v2.rs:5791-5797` field ordering** — placing `detail` before
   `url` is a valid ordering choice but reverses the natural
   "primary identifier first" convention used elsewhere in the
   file (e.g., `LocalImage` has `path` historically first). The
   schema-fixture diff shows the JSON output orders properties
   alphabetically anyway, so the struct field order is purely
   readability — but consistency with the surrounding code would
   be `url`/`path` first and `detail` second. Mechanical to flip.

4. **`#[ts(optional = nullable)]` in v2 vs. `#[ts(optional)]` in
   core** (`user_input.rs:28` vs. `v2.rs:5793`) — the difference
   produces TS types `detail?: ImageDetail` vs. `detail?:
   ImageDetail | null`. Both deserialize identically from the
   JSON (`null` and missing both unwrap to `None` Rust-side via
   `serde(default)`), but the inconsistency means a TS consumer
   typing against the core schema vs. the v2 schema sees two
   different shapes. Pick one and stick with it (the v2 form is
   probably right since the JSON Schema is `anyOf [ImageDetail,
   null]`).

5. **22 regenerated schema fixtures** committed as part of the
   PR — fine, since the README at
   `app-server-protocol/README.md` (presumably) calls out the
   `write_schema_fixtures` command as the canonical regen path,
   but this PR doesn't include a CI guard that asserts the
   committed fixtures match the current code. Worth confirming
   `schema_fixtures_match_generated` (referenced in the PR body's
   validation list) is wired into CI on every PR — otherwise a
   future field addition that forgets to regen will silently ship
   a schema/code drift.

6. **`local_image_content_items_with_label_number` retained as a
   thin back-compat wrapper at `:1050-1062`** — the wrapper
   exists for callers outside this PR. If those callers all live
   in the codex tree, the wrapper is dead weight and the call
   sites should be migrated to the `_and_detail` form (passing
   `DEFAULT_IMAGE_DETAIL` explicitly). If the function is part
   of a public API surface, it needs a `#[deprecated]` attribute
   pointing to the new form.

7. **`analytics_client_tests.rs:230-232,348-350`** adds
   `detail: None` to fixtures inside two `sample_*_request`
   builders — fine for compilation but worth at least *one* test
   request fixture in this file with a non-`None` `detail` so
   analytics serialization of the new field is also verified end-
   to-end, not just at the `models.rs` unit-test layer.

## Verdict rationale

The plumbing shape is correct (typed gate at the user input,
lossless forwarding through every intermediate representation,
strict-additive on-wire change, load-bearing
`prompt_image_mode_for_detail` branch for the `Original`-bypasses-
resize semantic), test coverage is dispositive on the most
important property (round-trip preservation through v2 and the
data-URL → InputImage conversion), and the schema-fixture regen
is consistent across all 16 envelope types. Worth a doc comment
on the resize-policy branch, parametrized coverage of all four
detail tiers, and a one-time normalization of `#[ts(optional)]` vs.
`#[ts(optional = nullable)]` so TS consumers see one shape across
core and v2.

`merge-after-nits`
