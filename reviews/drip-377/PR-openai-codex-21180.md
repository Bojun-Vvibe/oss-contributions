# openai/codex #21180 — Make turn diff tracking operation backed

- Head SHA: `f84c4eb7390c88de207301f5024a8f04a545560b`
- Diff: +1050 / -898 across 15 files
- Largest churn: `codex-rs/core/src/turn_diff_tracker.rs` (+233/-387) and its test sibling (+283/-380)

## Findings

1. Architecturally sound rewrite — the previous turn-diff tracker walked the filesystem comparing pre- and post-turn snapshots, which fights the upcoming Code-Cell-Agent file-system isolation (per the PR body: "For the CCA file system isolation push"). The new implementation is operation-backed: each `apply_patch` invocation now feeds an `AppliedPatchDelta` into the tracker via the new struct introduced at `codex-rs/apply-patch/src/lib.rs:184-200` (`AppliedPatchDelta { changes: Vec<AppliedPatchChange>, exact: bool }`). The `exact: bool` flag is the load-bearing detail — it captures whether the diff is byte-perfect (used for tracker-trustable cases) vs a synthesized fallback (used when the destination was unreadable, e.g. binary or symlink targets).
2. The `original_content` field added to `ApplyPatchFileUpdate` (visible in the test fixtures at `apply-patch/src/invocation.rs:711` and `:750`) is what enables move-overwrite cases to render correctly without the FS snapshot. The previous tracker inferred original-side content by re-reading the destination file before patching; with operation-backing, that read happens *as part of the patch operation* and is captured in the struct, eliminating a TOCTOU window where another process could mutate the destination between snapshot and patch.
3. Two new regression tests at `apply-patch/src/invocation.rs:862-916` — `test_unreadable_destinations_still_verify` (binary file + move-to-binary-destination) and `test_delete_symlink_still_verifies` — pin the previously-fragile cases where the old FS-walking tracker would fail or produce garbage. These are exactly the tests you want for an operation-backed rewrite.
4. The `pub use AppliedPatchDelta` is exported and consumed by `core/src/tools/events.rs` (+67/-19) — the public-API surface widening. Concern: the PR body's caveat — *"This takes the assumption that no 3P services rely on the output format of `apply_patch`"* — deserves an explicit owner sign-off in the PR before merge, because the unified-diff string format emitted by `unified_diff_from_chunks` is consumed by external tooling (downstream telemetry pipelines, CI report parsers) per the existing event payloads. If the diff string format changes verbatim across this PR, that's a silent breaking contract.

## Verdict

**Verdict:** needs-discussion
