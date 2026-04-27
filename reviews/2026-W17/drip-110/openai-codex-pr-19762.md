# openai/codex PR #19762 — make auth loading async (plumbing-only)

- **PR**: https://github.com/openai/codex/pull/19762
- **Author**: @efrazer-oai
- **Head SHA**: `a20884a8515d5ca5a75d3c1e5bd84e4e3e46cb1e`
- **Size**: +291 / −203 across 24 files
- **Stated scope**: pure plumbing — no change to AgentIdentity decode, runtime-id allocation, or JWT verification.

## Summary

Converts the synchronous auth-loading surface (`AuthManager::shared_from_config(...)` and `AuthManager::reload()`) to `async` and updates every call site to `.await`. The change touches 24 files across `app-server`, `chatgpt`, `cli`, `cloud-requirements`, `cloud-tasks`, `core`, `exec`, `login`, `mcp-server`, `models-manager`, and `tui`. Reason for going async (per PR description): some auth sources now need async work that synchronous helpers couldn't accommodate.

## Verdict: `merge-after-nits`

This is the kind of mechanical refactor where the bug surface is "did you miss a call site?" not "is the diff correct?". Spot checks of the diff confirm:

- All four `auth_manager.reload()` call sites in `app-server/src/codex_message_processor.rs` (lines 1356, 1507, 1615, 1751) became `.reload().await`.
- All three `AuthManager::shared_from_config(...)` call sites in `app-server/src/{in_process.rs:392,lib.rs:471,633,714}` got `.await` appended.

The PR description explicitly disclaims behavior change. The thing I'd want to see before "merge as is" is one of:

1. A grep proof in CI that no `AuthManager::shared_from_config` or `auth_manager.reload()` call site was missed (i.e. a `cargo check --all` is necessary but not sufficient — a deny-list lint would be stronger).
2. A test that constructs `AuthManager` with a slow (e.g. `tokio::time::sleep`-based) async auth source and asserts that the existing tests still pass without deadlock — this guards against any caller that's holding a non-`Send` guard across the new `.await` point.

## Specific references

- `codex-rs/app-server/src/codex_message_processor.rs:1356, 1507, 1615, 1751` — four `auth_manager.reload().await` conversions. All four are inside `async fn` already; no signature changes propagate up. Good.
- `codex-rs/app-server/src/in_process.rs:392-394` — `AuthManager::shared_from_config(args.config.as_ref(), args.enable_codex_api_key_env).await`. The surrounding function (`start_uninitialized`) was already async; minimal blast radius.
- `codex-rs/app-server/src/lib.rs:471, 633, 714` — three more `.await` insertions. The call at `:471` is inside `replace_cloud_requirements_loader`'s caller chain — verify that `replace_cloud_requirements_loader` doesn't itself synchronously call back into auth, otherwise the change just moves the blocking work one level outward.
- The diff also touches `chatgpt/src/{chatgpt_client,connectors}.rs`, `core/src/{connectors,prompt_debug}.rs`, `cli/src/{login,main}.rs`, and a long tail of `tests/suite/...`. The tests changing matters: it implies there's at least one runtime that previously constructed `AuthManager` from a `#[test]` (sync) context that now needs `#[tokio::test]`. A reviewer should verify no test silently switched from sync to "async-but-unawaited-Future-discarded" — easy mistake when a refactor is this wide.
- Stack context: per the PR body, this is PR 1 of a 3-PR stack — eager-load AgentIdentity runtime (#19763) and JWKS-verified AgentIdentity JWTs (#19764) follow. That ordering is correct: you can't eagerly load an async-fetched JWKS unless `AuthManager` construction can `.await`.

## Nits

1. Add a `cargo deny` rule (or a CI grep) that fails if `shared_from_config(` appears without `.await` on the same statement, to prevent regressions on backports.
2. Worth one Loom-style or `tokio::time::sleep`-injecting test that proves no caller holds a sync mutex across the new `.await` (the kind of bug that only shows up under load).
3. Confirm none of the touched test files dropped a `.await` and got a "future not awaited" warning suppressed by a top-level `#[allow(unused_must_use)]`. A quick `git grep -n 'allow(unused_must_use)'` on the diffed files would settle it.

## What I learned

Wide async-conversion PRs are easier to review when paired with a CI grep that *requires* the new shape. Otherwise reviewers fall back to "did I see all 24 file diffs scroll past" which is a notoriously poor bug detector. The 3-PR stack ordering here (`async surface → eager runtime load → JWKS verify`) is also a textbook example of "do the boring plumbing in its own PR so the interesting PR's diff is small enough to actually review the security claims."
