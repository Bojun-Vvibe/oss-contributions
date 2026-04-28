# PR #20086 — Fix plugin list workspace settings test isolation

- **Repo:** openai/codex
- **Link:** https://github.com/openai/codex/pull/20086
- **Author:** canvrno-oai
- **State:** OPEN
- **Head SHA:** `8a4864223738bdf9388e22eaf3f8d9006edc2652`
- **Files:** `codex-rs/app-server/tests/common/mcp_process.rs` (+9/-0), `codex-rs/app-server/tests/suite/v2/plugin_list.rs` (+18/-2)

## Context

Plugin-list tests in the app-server suite were leaking the host developer's `$HOME`/`$USERPROFILE` into the test process, which meant whatever marketplaces/plugin configs were installed on the developer's machine could leak into assertions about the plugin list. That's the canonical "passes on CI, fails on my laptop (or vice versa)" hermeticity bug.

## What changed

A new test helper in `mcp_process.rs` combines the existing managed-config isolation pattern with custom env overrides — so the test author can both isolate the codex managed config dir *and* shove deterministic values into env vars that the app-server reads. In `plugin_list.rs`, the workspace-settings test now uses that helper to set `HOME` (and the Windows equivalent `USERPROFILE`) to a tempdir, so the plugin-resolution code can't reach the developer's real home.

## Design analysis

This is the right move. Two observations:

1. **Cross-platform env vars.** Windows reads `USERPROFILE` rather than `HOME`. The PR description mentions both, which is the correct compatibility surface. Worth checking the helper sets *both* unconditionally rather than gating on `cfg(windows)` — it's safe to overwrite `HOME` on Windows even if it's unused, but if you only set one and a test runs on a developer's Windows box that has `HOME` set (say from msys2/git-bash), you'd still leak. Belt-and-suspenders: set both on every platform.

2. **Why a new helper rather than extending the existing one?** The PR adds 9 lines to `mcp_process.rs` for the new helper. If the existing managed-config helper is the natural composition point, adding an `env_overrides: Option<HashMap<...>>` parameter would have been a smaller diff. New helpers tend to grow into "the one I happened to need" siblings. Worth asking whether the existing helper can be extended instead; if not, the comment on the new helper should explain why composition wasn't viable.

## Risks

The changed test (line ~ in `plugin_list.rs`, in the workspace-settings test) is the only consumer right now. If the team intends every plugin/marketplace test to use this isolation pattern, follow-up PRs should sweep the rest. Until they do, the suite has mixed hermeticity, which is a maintenance smell. This is a "single PR for a single test" ratio that's fine here but worth flagging in the PR body so future readers know the migration is intentionally incremental.

The only correctness risk I see is if any code path under test caches the result of reading `HOME`/`USERPROFILE` at *startup* of the test binary (before the helper runs). That would defeat the override. Standard Rust test layout makes this unlikely (each `#[test]` fn runs in the same process but state is constructed inside the test) but worth a sanity check that no `lazy_static!` or `OnceCell` is grabbing the host home at module init.

## Verdict

**Verdict:** merge-as-is

Small, well-scoped hermeticity fix. The two design questions above are nice-to-haves but don't block merge — they can be addressed in follow-up if the helper proliferates.

---

*Reviewed by drip-155.*
