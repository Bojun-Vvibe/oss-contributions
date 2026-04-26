# block/goose PR #8823 — chore(deps): bump tiktoken-rs from 0.6.0 to 0.11.0

- Repo: block/goose
- PR: #8823
- Head SHA: `4f15ba93`
- Author: @app/dependabot
- Diff: +11/-38 across 2 files (`Cargo.lock`, `crates/goose/Cargo.toml`)

## What changed

Five-minor `tiktoken-rs` bump (`0.6.0` → `0.11.0`) on a single line in `crates/goose/Cargo.toml:121`. The lockfile churn at `Cargo.lock` is mostly cleanup from the transitive bump: the duplicate `bit-set 0.5.3` + `bit-vec 0.6.3` pair (previously pulled in by the old `fancy-regex 0.13.0` that `tiktoken-rs 0.6.0` depended on) collapses to a single `bit-set 0.8.0` / `bit-vec 0.8.0` pair, and `fancy-regex 0.13.0` is removed entirely in favor of `0.14.0`.

## Specific observations

- The five-minor jump (`0.6` → `0.7` → `0.8` → `0.9` → `0.10` → `0.11`) crosses several breaking-change boundaries that dependabot will not flag. Notably 0.7 split `cl100k_base` / `o200k_base` enums; 0.8 changed `CoreBPE::encode` return type from `Vec<usize>` to `Vec<u32>` for memory; 0.10 introduced the `tokenizer` cargo feature that gates the higher-level `get_completion_max_tokens` / `num_tokens_from_messages` helpers — the diff does not show a `features = […]` annotation in `crates/goose/Cargo.toml:121`, so any goose code calling those helpers will get a `tokenizer feature not enabled` compile error or, worse, a missing-symbol link error at runtime. Run `git grep -E 'tiktoken_rs::(num_tokens|get_completion|get_chat)'` across `crates/goose*/` before merge.
- The `Cargo.lock` consolidation is genuinely good — the old tree pulled both `bit-set 0.5.3` (via `fancy-regex 0.13.0`) and `bit-set 0.8.0` (via `regex-cli`), now collapsed to a single `bit-set 0.8.0` entry at lockfile-line block around `1278-1283`. Same story for `fancy-regex` itself going `0.13.0` (deleted at lock lines `3508-3517`) → only `0.14.0` (lines `3522-3530`) remaining. Net effect is a smaller compile graph and one fewer duplicate-crate-name warning from `cargo deny`.
- The unrelated `itertools 0.13.0` → `0.14.0` change at `Cargo.lock:7358` (proc-macro side) is a transitive artifact of the bump and not a direct goose change — fine, but worth confirming nothing in `crates/goose/src/**` was relying on `itertools 0.13` behavior (e.g., `tuple_windows` slice-vs-iterator differences).

## Verdict

`request-changes`

## Rationale

Five-minor pre-1.0 tokenizer bump on a single Cargo.toml line is high-risk without a manual API audit. Run `cargo grep` across goose for `tiktoken_rs::num_tokens_from_messages` / `tiktoken_rs::get_completion_max_tokens` / `CoreBPE::encode` callers, confirm whether the new `tokenizer` feature needs to be enabled (`tiktoken-rs = { version = "0.11.0", features = ["tokenizer"] }`), kick the full CI matrix including any LLM-token-budget tests, and post-merge, validate that `num_tokens_from_messages` returns the same counts for at least gpt-4o + gpt-4o-mini before cutting a release.
