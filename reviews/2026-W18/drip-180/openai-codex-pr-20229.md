# openai/codex#20229 — PoC: codex-kernel

- **PR:** https://github.com/openai/codex/pull/20229
- **Author:** @bolinfest
- **Head SHA:** `c0845d39` (full: `c0845d39e0380adc2d474cb192edb7cab91a60ad`)
- **Size:** +1104/-549 across 19 files
- **Status:** explicitly marked PoC ("draft PR is a proof of concept …")

## What this does

Carves a new `codex-kernel` workspace crate (`codex-rs/kernel/`, 5 source
files, ~700 LOC of new code) out of `codex-core`. The split moves three
things into the kernel:

1. **`Prompt` struct** (`kernel/src/prompt.rs`, 257 LOC) — moved verbatim
   from `core/src/client_common.rs`. The deletion in `client_common.rs` is
   ~159 LOC, replaced with a single `pub use codex_kernel::Prompt;`
   re-export. The reserialize-shell-output helper
   (`reserialize_shell_outputs`) and its tests
   (`client_common_tests.rs:166-228`) move along with `Prompt` because
   `Prompt::get_formatted_input()` calls them.

2. **Retryable outer sampling loop** (`kernel/src/sampling.rs`, 170 LOC) —
   pulled from `core/src/session/turn.rs` (which loses 163 LOC, gains 22
   LOC of host-side glue). The new home for "model-emitted-a-tool-call,
   attempt the complementary output item" policy.

3. **Kernel tool contract** (`kernel/src/tools.rs`, 226 LOC) — defines
   `KernelToolExecutor`, `KernelToolCall`, `CompletedResponseItem`. Host
   side (`core/src/tools/parallel.rs` +90/-46) implements the executor.

The new crate is added to `Cargo.toml` workspace members, gets a
workspace-level dep alias `codex-kernel = { path = "kernel" }`, and is
pulled into `codex-core/Cargo.toml`.

## What I like

- **The seam choice is real.** Splitting `Prompt` + sampling-loop +
  tool-executor-contract is the smallest surface that lets you write a
  non-`codex-core` host. The PoC doesn't try to hide tool implementations
  behind the kernel — those stay in `core/src/tools/runtimes/` — only the
  loop policy and the dispatch contract move.
- **`stream_events_utils.rs` shrinks from ~94 LOC to 12 LOC** — that's the
  smoking gun that the kernel is taking the right responsibility (model
  item classification + dispatch).
- **`Prompt` re-exported with the same name** — `pub use codex_kernel::Prompt;`
  in `core/src/client_common.rs` means the public surface of `codex-core`
  doesn't break for downstream crates that imported
  `codex_core::client_common::Prompt`.

## Concerns

1. **Visibility regression in the move.** Original `Prompt` had several
   `pub(crate)` fields: `tools`, `parallel_tool_calls`. After moving to
   `codex-kernel` as a separate crate, `pub(crate)` becomes "visible
   inside the *kernel* crate", which means `codex-core` can no longer
   touch these fields directly without the kernel exposing accessors.
   Either the fields became `pub` (silent surface expansion), or the
   kernel grew accessors that aren't shown in the snippet I have. Worth
   a `git diff -- 'codex-rs/kernel/src/prompt.rs'` audit to confirm
   visibility didn't get widened to `pub`.

2. **`get_formatted_input(&self) -> Vec<ResponseItem>` was `pub(crate)`
   in `client_common.rs:38`.** Same question — did the move force
   widening to `pub`? If yes, that's a real public-API addition that
   should be called out in the PR description.

3. **Test relocation, not duplication.** `client_common_tests.rs` loses
   `reserializes_shell_outputs_for_function_and_custom_tool_calls`
   (~63 LOC). I assume it now lives in `kernel/src/prompt.rs` as a
   `#[cfg(test)] mod tests`, but the file list shows
   `kernel/src/prompt.rs` at exactly 257 additions, which is suspiciously
   close to the original `Prompt` LOC. If the test didn't move, regression
   coverage was silently dropped.

4. **The `app-server/tests/suite/v2/turn_start.rs` change is unrelated
   to the split.** It rewrites the skills-warning assertion from a
   hardcoded `7 additional skills` string to a parsed-count assertion
   accepting both singular and plural suffixes. That's a sensible
   robustness fix but it's a separate PR's concern — including it here
   makes the kernel-extraction diff harder to read. The PR description
   acknowledges this ("Updated the app-server skills warning assertion to
   validate the warning structurally instead of depending on the current
   number of installed skills") but the rationale is still weak: why did
   the kernel split change the installed-skill count? If it didn't, the
   test fix doesn't belong in this PR.

5. **`BUILD.bazel` exists but no `Cargo.lock` audit shown.** The new
   `codex-kernel` crate adds `pretty_assertions`, `rand 0.9.3`, `serde`,
   `serde_json`, `tokio`, `tokio-util`, `codex-protocol`, `codex-tools`
   to its dep tree. No new transitive deps appear in the lockfile diff
   beyond the kernel package itself, which is good — but worth one more
   `cargo tree -d` pass before promoting from PoC.

## Verdict

**`needs-discussion`** — the seam is the right one and the LOC math
checks out, but as a PoC this needs a design conversation about (a)
whether `Prompt` field visibility silently widened, (b) whether the
unrelated turn_start test change should be split out, and (c) whether
`codex-kernel` should also own the request/response types or stay
purely loop-policy. The PR explicitly invites that discussion ("less
'final API polish' and more 'can this boundary carry the existing
app-server behavior'"), so this verdict matches the author's intent.

## Nits

- Drop the `app-server/tests/suite/v2/turn_start.rs` change into its
  own PR — its merit is independent and it would land faster.
- Add a `kernel/README.md` describing the boundary contract: what
  belongs in kernel vs host. Given this is a brand-new crate, the
  doctrine is at its most malleable now.
- Audit `pub(crate)` → `pub` widenings introduced by the move and
  call them out in the PR description.
