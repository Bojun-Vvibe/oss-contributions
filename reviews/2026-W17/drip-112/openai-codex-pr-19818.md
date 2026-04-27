# openai/codex PR #19818 — chore: split memories into `read` / `write` crates (part 1)

- **PR**: https://github.com/openai/codex/pull/19818
- **Author**: openai contributor
- **Head SHA**: `3c5ad4d09132`
- **Size**: +436 / −267
- **Files**: 39 files; introduces `codex-rs/memories/read/` and `codex-rs/memories/write/` crates, moves `core/src/memories/{citations,citations_tests,extensions,extensions_tests,control}.rs` into them, splits `usage.rs` between core and the new read crate, adds `BUILD.bazel` files, and rewires `core` to depend on both.

## Summary

Pure structural move: the existing `codex-core::memories` module is being carved into two new sibling crates so the "read" surface (citations, prompts, usage metering for retrieval) and the "write" surface (extensions registry, control flow, write-side state) can evolve, be feature-gated, and be tested independently. `Cargo.toml` workspace gets `memories/read` + `memories/write` members and `core/Cargo.toml` adds the two as workspace dependencies. `core/src/lib.rs` removes the `memories` module declaration.

## Verdict: `merge-after-nits`

This is a "part 1" refactor PR with no behavior changes — exactly the kind of large mechanical move that should land quickly, before conflicts pile up. The risk is in the boundary: did anything that *should* have stayed in `core` accidentally move out, and did the new crate dependency graph (read/write each pulling protocol/state/utils) introduce a cycle or lengthen the build? Those are the two things to verify before merge.

## Specific references

- `codex-rs/Cargo.toml:48-49,157-158` — adds `"memories/read"` and `"memories/write"` as workspace members and `codex-memories-read = { path = "memories/read" }` / `codex-memories-write = { path = "memories/write" }` as workspace dependencies. Standard pattern; matches the other `codex-*` sub-crates already in the file.
- `codex-rs/Cargo.lock:2370-2374, 2923-2956` — the lockfile grows two new package entries:
  - `codex-memories-read` depends on `codex-protocol`, `codex-shell-command`, `codex-utils-absolute-path`, `codex-utils-output-truncation`, `codex-utils-template`, `pretty_assertions`, `tempfile`, `tokio`. Note: **no `codex-state`, no `codex-models-manager`** — read side is correctly free of the heavier write-side dependencies.
  - `codex-memories-write` adds `anyhow`, `chrono`, `codex-git-utils`, `codex-models-manager`, `codex-state`, `tracing`, `uuid` on top. The presence of `codex-models-manager` and `codex-state` in `write` only is the right asymmetry — write needs to mutate session state and resolve model context, read does not.
- `codex-rs/core/src/lib.rs` — removes the `pub mod memories;` declaration. The `core` crate now re-exports from the two new crates instead of owning the module tree directly.
- `codex-rs/core/src/config/mod.rs:4` — drops `use crate::memories::memory_root;`. The `memory_root` symbol must now be re-exported from one of the two new crates and re-imported here. Verify in CI that this compiles end-to-end (it's the kind of subtle break a fresh `cargo check --workspace` would catch but a partial build might miss).
- File-move references (the `diff --git a/codex-rs/core/src/memories/X b/codex-rs/memories/{read|write}/src/X` lines): `citations.rs` + `citations_tests.rs` + `prompts.rs` + `usage.rs` (read-flavor) + `read_path.md` template land in `memories/read/`; `control.rs` + `extensions.rs` + `extensions_tests.rs` land in `memories/write/`. The split is on the read-vs-write axis, not on data-vs-policy, which is the right cut for future feature-gating.
- The remaining `core/src/memories/{mod,phase1,phase2,tests,usage}.rs` (now `core/src/memory_usage.rs` per the rename in Cargo.lock context) keep the orchestration glue inside `core`. That's correct — the orchestrator owns the lifecycle, the two new crates own the domain surfaces.

## Nits

1. The PR title says "part 1" but the description is just "Extract memories into 2 different crates" with no roadmap for parts 2+. Add a one-paragraph plan to the PR body: what stays in `core::memories` and what's deferred to part 2 (presumably the orchestration `phase1.rs` / `phase2.rs` themselves)? This helps reviewers and the next refactor PR.
2. The new crates have empty `BUILD.bazel` stubs alongside Cargo manifests. If the project's Bazel build is gated by CI, ensure those `BUILD.bazel` files are non-empty enough to compile under `bazel build //codex-rs/memories/...`. Otherwise this PR will silently break the Bazel path.
3. `pretty_assertions`, `tempfile`, `tokio` are pulled into both crates as dev-or-prod deps. If they're only used in tests, mark them `[dev-dependencies]` to keep the production binary lean.
4. The fact that `read` does NOT depend on `state`/`models-manager` is a useful invariant — consider asserting it with a `#[deny(...)]` or a doc comment on the read crate's `lib.rs` so a future PR doesn't drift it back to "read knows about everything".

## What I learned

Splitting a `memories` module across the read/write axis (rather than the more common data/policy or sync/async axis) is a strong signal that the team plans to feature-gate or sandbox the write path separately from the read path — likely so memory writes can be tested with mock state while memory reads stay cheap and side-effect-free. The dependency asymmetry in `Cargo.lock` (write pulls `state`+`models-manager`, read does not) is the cleanest way to enforce this — anyone trying to add `state` to the read crate will be visible in a future Cargo.lock diff. This is much cheaper than a runtime guard.
