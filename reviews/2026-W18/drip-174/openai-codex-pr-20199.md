# openai/codex #20199 — feat: codex journal

- **PR:** https://github.com/openai/codex/pull/20199
- **Title:** `feat: codex journal`
- **Author:** jif-oai
- **Head SHA:** 63402cbc3bf74528e0151505a0686ee12a16dc9e
- **Files changed:** 44 (new `codex-rs/journal/` crate + integration into `core` and `thread-store`), **+3329 / −1010**
- **Verdict:** `needs-discussion`

## What it does

Introduces a new first-class `codex-journal` crate that owns durable,
classifiable conversation history, and migrates the in-memory
`context_manager::history` representation into it. Roughly:

1. **New crate `codex-rs/journal/`** (lib.rs, journal.rs +345, history/
   submodule with classify.rs +133, estimate.rs +174, normalize.rs
   +105/−83, tests.rs +286, transform.rs +92, plus prompt_view.rs +33,
   error.rs, tests.rs +349). New protocol module
   `codex-rs/protocol/src/journal.rs` (+383) defines the wire types.
2. **`core` migration:** `context_manager/history.rs` is **deleted
   entirely (−726 lines)** and replaced by a thin `state/history.rs`
   shim (+314) plus rewrites of every consumer (`compact.rs`,
   `compact_remote.rs`, `arc_monitor.rs`, `realtime_context.rs`,
   `session/handlers.rs`, `session/mod.rs` +135/−66,
   `session/rollout_reconstruction.rs`, `session/turn.rs`,
   `state/session.rs`, `agent/control_tests.rs`).
3. **`thread-store` integration:** new
   `thread-store/src/journal_writer.rs` (+355) plus `Cargo.toml` and
   `lib.rs` wiring.
4. The migration touches the public-ish API: `history.raw_items()` on a
   value becomes `session_history::raw_items(&history)` on a reference,
   `history.record_items(...)` becomes
   `session_history::record_items(&mut history, ...)`,
   `history.for_prompt(...)` becomes `session_history::for_prompt(&history, ...)`,
   `history.remove_first_item()` becomes
   `session_history::remove_first_item(&mut history)`. Test code in
   `agent/control_tests.rs:36-46` introduces a `JournalTestExt` adapter
   trait so existing tests can call `.raw_items()` on the new `Journal`
   type.

## Why I'm flagging it for discussion (not nit-picking)

The PR body is empty. A 3329-add / 1010-delete PR that **deletes a
726-line context-manager core module** and replaces it with a new
top-level crate, with **no PR description, no migration notes, no
verification claim, and no listed test command,** is not in a state
where a 17-minute drive-by review can make a call.

Substantive concerns that require an answer from the author before
merge:

1. **Why a new crate, not a `core::journal` module?** A new top-level
   crate (`codex-rs/journal/`) adds a crate-graph entry to every
   downstream binary (Cargo.lock entry at lines ~2816-2832 lists 11
   transitive deps including `image`, `base64`, `indexmap`, `tracing`,
   `thiserror`, `pretty_assertions`, `tempfile`). If this is going to be
   reused by `mcp`, `app-server`, `tui`, etc. independently of `core`,
   say so. If only `core` and `thread-store` consume it, it should
   probably be a `core/src/journal/` submodule until a third consumer
   lands.
2. **Free-function migration is a regression in ergonomics.** The
   pattern `history.raw_items()` → `session_history::raw_items(&history)`
   appears in `arc_monitor.rs:228`, `compact.rs:163,180,254`,
   `agent/control_tests.rs:811,922-940`, etc. (probably 30+ call sites
   across the diff). This is the inverse of how `Journal` should be
   used as a domain type — methods belong on the type. The
   `JournalTestExt` trait at `agent/control_tests.rs:36-46`, which only
   exists to forward `.raw_items()` from the trait back to the free
   function, is evidence of the friction. Either add inherent methods on
   `Journal` (a 50-line wrapper) or explain why free functions are the
   right shape.
3. **The new `codex-protocol::journal` module is +383 lines with no
   doc-comments quoted in the diff hunks I sampled.** A wire-format
   surface this large needs at minimum a top-level "stability:
   experimental / stable" marker and a serde versioning story. A
   silently-shipped wire format becomes load-bearing the moment a
   downstream binary starts persisting it.
4. **No test claim.** A migration of this size without `cargo test
   --workspace` confirmation in the PR body is not safe to land. The
   only test changes I can see are mechanical adapter-trait rewrites in
   `agent/control_tests.rs` and a new `state/history_tests.rs` (+113
   lines) — but the *deleted* `context_manager/history.rs` presumably
   had its own test surface that needs to be confirmed migrated, not
   dropped.
5. **`config.schema.json` doc-comment-only churn** (`tool_output_token_limit`:
   "stored in the context manager" → "stored in thread history") is the
   cosmetic tip of a much larger conceptual rename (context manager →
   thread / journal). The PR doesn't acknowledge this rename in its
   title or body. Future blame digs for "where did the context manager
   go" will find nothing.

## What's good (so far as I can tell)

- The classify/estimate/normalize/transform module split inside
  `journal/history/` (`classify.rs:+133`, `estimate.rs:+174`,
  `normalize.rs:+105/−83`, `transform.rs:+92`) is the right shape for a
  history pipeline — separating "what kind of item is this" from "how
  big" from "how do we canonicalize" from "how do we mutate."
- The migration appears mechanical and consistent: every
  `history.X(...)` site is rewritten to `session_history::X(&history, ...)`
  with no behavioral change visible in the hunks I read. That's the
  safest possible refactor shape *if* it's covered by tests.
- The new `state/history.rs` shim (+314) keeping the old `crate::state::history`
  path alive avoids a giant `use` rewrite across the rest of the tree.

## What I'd need to flip this to `merge-after-nits`

1. PR body filled in: motivation, scope, what's now in `journal/` vs
   what stayed in `core`, why a separate crate, what's the wire-format
   stability promise.
2. `cargo test --workspace --no-default-features` and
   `cargo test --workspace --all-features` results pasted, plus
   `cargo clippy --workspace --all-targets -- -D warnings` clean.
3. A justification for the free-function call shape, OR inherent
   methods on `Journal` so call sites don't need
   `session_history::foo(&history, ...)` and the `JournalTestExt`
   adapter trait disappears.
4. Either a follow-up split into `[1/N]` PRs (crate skeleton; protocol
   types; core migration; thread-store integration) or a serial
   walkthrough of the diff in the PR body explaining what to read in
   what order.

## Verdict

`needs-discussion` — this is a directionally-reasonable refactor that
can't be reviewed responsibly in its current shape. Empty PR body +
unstated test results + a 1448-line crate skeleton + a 726-line core
deletion + a free-function ergonomic regression + a silent wire-format
introduction is too many simultaneous decisions for one PR. Ask the
author to either (a) split into the obvious slices and write descriptions
or (b) post a detailed PR body and test-output dump and we'll re-review.
