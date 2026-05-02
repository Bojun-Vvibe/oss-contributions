# Review: openai/codex #20670 — [codex-exec] Make review-policy params test hermetic

- Repo: openai/codex
- PR: #20670
- Head SHA: `6d7bbc5a3e22fb88dd6377008077a8af6e8759e7`
- Author: evawong-oai
- Size: +1 / -0 across 1 file

## What it does
Adds `LoaderOverrides::without_managed_config_for_tests()` to a single config
builder in the review-policy params test so the test no longer reads any
host-machine managed config layer that may exist in CI or developer machines.

## File-level notes
**`codex-rs/exec/src/lib_tests.rs` @ L347 (head `6d7bbc5`)**
```rust
let config = ConfigBuilder::default()
    .codex_home(codex_home.path().to_path_buf())
+   .loader_overrides(LoaderOverrides::without_managed_config_for_tests())
    .harness_overrides(ConfigOverrides {
        approvals_reviewer: Some(ApprovalsReviewer::User),
```
- One-line hermeticity fix targeting a single test
  (`thread_start_params_include_review_policy_when_review_policy_is_manual_*`).
  Pattern matches what other hermetic tests in this crate already do.
- The `LoaderOverrides::without_managed_config_for_tests` constructor name
  signals intent clearly; no risk to production code paths.
- A small follow-up worth tracking: there are likely sibling tests in the
  same file constructing `ConfigBuilder::default()` without this override.
  A grep + scoped fix would prevent the same flake recurring elsewhere, but
  is not required to land this PR.

## Risks
- None. Test-only change, additive.

## Verdict: `merge-as-is`
Trivial, correct, scoped to test code. Land it.
