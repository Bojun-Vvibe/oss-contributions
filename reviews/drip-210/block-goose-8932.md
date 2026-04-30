# block/goose#8932 — break up acp/server.rs

- PR: https://github.com/block/goose/pull/8932
- Head SHA: `58d57e56876f58bbcc6973573d26d9c93b991989`
- Author: jamadeo
- Files: 12 changed, +2072/−1720 (net +352 from boilerplate around module
  splits; near-zero net code change). `crates/goose/src/acp/server.rs`
  −1720 / +12.

## Context

`crates/goose/src/acp/server.rs` had grown to the point where every custom
ACP method handler (extensions, tools, sessions, providers, dictation,
secrets, sources, resources, custom-dispatch dispatch) was implemented in
a single file. The PR description is candid: "every custom method handler
exists here, when they should be broken up. This is a very basic version:
just moving the implementations to different files."

## Design

Pure file-split refactor. The diff at `crates/goose/src/acp/server.rs`
adds a module declaration block (`:69-80`):

```rust
mod config;
mod custom_dispatch;
mod dictation;
mod dispatch;
mod extensions;
mod providers;
mod resources;
mod secrets;
mod sessions;
mod sources;
mod tools;
```

…and removes the in-file implementations of each — visible in the diff at
e.g. the deletion of `inventory_entry_to_dto` (`:586-619`),
`provider_config_key_to_dto` (`:621-630`), `mask_secret_value` /
`config_value_to_string` / `provider_config_field_value` (`:636-690`),
`refresh_skip_reason_to_dto` / `refresh_plan_to_response` (`:692-720`),
the `OPENAI_TRANSCRIPTION_MODEL_CONFIG_KEY` block of dictation constants
(`:117-122`), and the very large `#[custom_methods] impl GooseAcpAgent`
block at `:2925-...` (where the on_add_extension / on_remove_extension /
on_get_tools / etc. handlers used to live).

The 11 new files in `crates/goose/src/acp/server/`:

- `config.rs` (+36) — config-related helpers
- `custom_dispatch.rs` (+354) — the `#[custom_methods]` macro expansion
  surface (the largest of the new files; this is where the deleted
  on_add_extension/on_remove_extension/on_get_tools impls now live)
- `dictation.rs` (+406) — transcribe_with_provider / whisper / per-provider
  helpers + the `OPENAI_TRANSCRIPTION_MODEL` constants
- `dispatch.rs` (+397) — primary RPC dispatch
- `extensions.rs` (+136) — `add_extension` / `remove_extension` glue
- `providers.rs` (+420) — `inventory_entry_to_dto`,
  `provider_config_key_to_dto`, `provider_config_field_value`,
  `refresh_skip_reason_to_dto`, `refresh_plan_to_response` — i.e., the
  provider-inventory DTO-conversion helpers
- `resources.rs` (+21) — small file, likely the resource-listing handler
- `secrets.rs` (+32) — `mask_secret_value`, `config_value_to_string`, the
  secret-handling utilities
- `sessions.rs` (+175) — session lifecycle handlers
- `sources.rs` (+65) — provider-source helpers
- `tools.rs` (+18) — tool-listing handlers

The +12 lines remaining in `server.rs` are exactly the `mod ...;`
declarations (12 module names = 12 lines, accounting for the extra blank
line). Nothing else was added to `server.rs` — confirmed by reading the
diff window: the removals dominate and the only additions are the module
declarations.

## What's right about this shape

- **No behavior change.** The `#[custom_methods]` macro just needs to
  see the impl blocks; whether they live in one file or eleven is
  irrelevant to the macro expansion as long as they're all in modules
  reachable from the same crate. The dispatch table the macro builds
  from `#[custom_method(AddExtensionRequest)]` annotations is built
  per-impl-block, so splitting one mega-impl into per-domain impl
  blocks is a no-op at the wire level.
- **Per-domain split, not per-handler split.** The new files are
  organized by *what they do* (providers, extensions, dictation,
  secrets), not by *which method they handle*. A naive split would
  produce one file per handler (`on_add_extension.rs`,
  `on_remove_extension.rs`, etc.) and lose the per-domain locality —
  the `mask_secret_value` / `config_value_to_string` helpers are
  conceptually one "secrets" concern even though they don't appear in
  any single handler. The chosen split keeps that locality.
- **No new pub-level surface.** Each new file is `mod xxx;` (private),
  so the only thing visible to crate consumers is what `server.rs`
  re-exports — same as before.

## Risks / nits

- **Diff is unreviewable line-by-line.** A 2k-line move-and-rename PR
  needs `git log --follow` discipline downstream, and `cargo check -p
  goose` is the only realistic correctness gate during review. The
  PR doesn't list a CI verification command in the description —
  worth confirming that the CI matrix (which presumably runs
  `cargo check` and `cargo test -p goose`) is green on the head SHA
  before merge.
- **Per-feature `#[cfg(feature = "local-inference")]` blocks** were
  removed from `server.rs` (visible in the deletion at `:13-17`
  removing the local-inference imports for `transcribe_local` and
  `whisper`). They presumably moved into `dictation.rs`. If the
  cfg-attr was lost in the move, `--no-default-features` builds will
  break. Worth a quick local `cargo check --no-default-features -p
  goose` before merge.
- **No tests touched.** Acceptable for a pure file-split, but it
  means there's no positive signal that, e.g., the
  `#[custom_method(AddExtensionRequest)]` annotation is still wired
  up correctly post-split. A single `cargo test -p goose
  test_acp_dispatch_known_methods` (or whatever the existing
  smoke test is named) would close that. If no such test exists,
  this is a good moment to add one — even a "list all dispatch
  table entries and assert the count is N" style test.
- **`use ...` imports at the top of the new files** aren't visible
  in this diff window. Reviewers should check that each new file's
  imports are minimal (no `use crate::*` star imports). Otherwise
  the split is mechanically right but semantically still
  "everything sees everything".

## Verdict

**`merge-after-nits`**

The split is correctly scoped (per-domain, not per-handler), preserves
the macro-driven dispatch shape, and adds no new public surface. The
nits are: (1) confirm `--no-default-features` build, (2) confirm CI
runs on head SHA and is green, (3) a one-line "what was here" comment
at the top of each new file would help the next reader understand the
domain boundary. None of these block merge in principle, but they're
the difference between "this lands clean" and "we revert next week
because Linux/no-features broke".

## What I learned

A 2k-line file-split PR is not actually a "review the diff" PR — it's
a "trust the test suite, sample 3 files, verify the module boundaries
make sense, and merge" PR. The right discipline for the *author* is
to split the move from any rename/refactor: this PR did exactly that
("just moving the implementations to different files"), so the diff
is genuinely a no-op at the AST level modulo the `mod` declarations.
The right discipline for the *reviewer* is to (a) confirm the file
list matches a sensible domain decomposition (it does — providers,
secrets, dictation, sessions etc. are real concerns), (b) sample one
of the larger new files to confirm it's not a kitchen-sink, and (c)
trust CI for everything else. The next step in this codebase is the
*hard* refactor — extracting per-domain `Service` traits so the
handler files don't all need to reach back into `GooseAcpAgent` for
state. This PR is the precondition for that.
