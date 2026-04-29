---
pr: openai/codex#20242
sha: 361f003f2915d67339cc83872b2e6a5b36d3cc2e
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# docs: discourage `#[async_trait]` and `#[allow(async_fn_in_trait)]`

URL: https://github.com/openai/codex/pull/20242
Files: `AGENTS.md`
Diff: 6+/0-

## Context

Codex has been progressively migrating from heap-allocating `async_trait`
boxed futures to native return-position-impl-trait-in-trait (RPITIT) with
explicit `Send` bounds — see referenced commit `3c7f013f9735` / PR #16630.
This doc-only PR codifies the convention in `AGENTS.md` so contributors
(human and agent) stop reaching for `#[async_trait]` or
`#[allow(async_fn_in_trait)]` as a shortcut.

## What's good

- The added block at `AGENTS.md:22-27` spells out the preferred trait
  shape literally: `fn foo(&self, ...) -> impl std::future::Future<Output = T> + Send;`.
  This is exactly the right level of specificity for a contributor doc —
  it's copy-pasteable and unambiguous about the `Send` bound, which is the
  load-bearing detail for cross-await-point usage in Tokio.
- The note that "implementations may still use `async fn foo(&self, ...) -> T`
  when they satisfy that contract" (`AGENTS.md:25`) is the correct
  permissive escape valve — it stops the rule from forcing impl-side
  rewrites that were already correct.
- Anchoring against `#[allow(async_fn_in_trait)]` explicitly
  (`AGENTS.md:26`) closes the "I'll just paper over the warning" loophole
  that this kind of guidance usually leaks through.
- Sits in the existing Rust subsection of `AGENTS.md` — natural place,
  no doc-architecture churn.

## Verdict reasoning

Pure doc clarification of an already-established convention with a
concrete example commit/PR reference for archaeology. Zero code risk,
zero CI risk. Land it.
