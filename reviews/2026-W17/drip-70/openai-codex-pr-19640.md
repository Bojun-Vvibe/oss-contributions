# openai/codex#19640 — [codex] remove responses command

- **Repo**: openai/codex
- **PR**: [#19640](https://github.com/openai/codex/pull/19640)
- **Head SHA**: `633fe00a9c54`
- **Author**: tibo-openai
- **Base**: `main`
- **Size**: +7 / −289, 4 files

## Context

The hidden `codex responses` subcommand was an internal escape hatch for
sending one raw Responses-API payload through Codex auth. The standalone
`codex-responses-api-proxy` long-running mode (still kept as a hidden
`ResponsesApiProxy` subcommand) covers the same use case in a more durable
way, so the one-shot variant is dead surface.

## Change

Three coordinated deletions:

1. `codex-rs/cli/src/main.rs:46` drops the `mod responses_cmd;` declaration
   and the two `use crate::responses_cmd::{ResponsesCommand,
   run_responses_command};` imports.
2. `codex-rs/cli/src/main.rs:163-167` removes the `Responses(ResponsesCommand)`
   `Subcommand` variant and its match arm at line ~1126 that did the
   `reject_remote_mode_for_subcommand(..., "responses")` plus
   `run_responses_command(...)` dispatch.
3. `codex-rs/cli/src/responses_cmd.rs` is deleted entirely.
4. `codex-rs/cli/Cargo.toml` drops the `codex-api` and `codex-model-provider`
   workspace deps that were only pulled in to satisfy `responses_cmd.rs`,
   and `codex-rs/Cargo.lock` is regenerated to match (two `dependencies`
   entries removed from the `cli` package block).

## Strengths

- Surgical: the `Cargo.toml` and `Cargo.lock` deltas are exactly the
  transitive pulls the deleted module owned. Nothing else in `cli/` was
  importing `codex-api` or `codex-model-provider`, so the dep removal is
  proven by the fact the build still passes (assuming CI is green on the
  PR — worth confirming).
- Hidden + internal-only subcommand → no external CLI contract to deprecate
  first. The `#[clap(hide = true)]` annotation in the original definition
  means no `--help` user ever saw it, so a straight delete is fine.
- Keeps the more general `ResponsesApiProxy` subcommand intact — the
  remaining hidden surface still serves debugging needs.

## Risks / nits

1. No release-notes / CHANGELOG entry. Even hidden internal subcommands
   sometimes get scripted by other internal tools; a one-line
   "Removed hidden `codex responses` subcommand; use
   `codex responses-api-proxy` instead" is cheap insurance. Confirm
   whether anyone in the openai/codex monorepo has CI scripts that shell
   out to `codex responses` — a quick `rg 'codex responses[^-]'` across
   sibling repos would prove it.
2. The two removed workspace deps (`codex-api`, `codex-model-provider`)
   are still built when other workspace crates depend on them, so this
   is a build-time dep-graph thinning of the `cli` crate only — not a
   workspace-wide simplification. Worth noting in the commit message
   so a future reader doesn't think those crates became unused.
3. If the Responses-API proxy mode ever needed a "send one payload and
   exit" shortcut again, the right resurrection is a `--once` flag on
   the proxy, not a new subcommand. Worth dropping that as a comment in
   the PR for posterity.

## Verdict

**merge-as-is** — clean, mechanical, dep-coherent removal of dead hidden
surface. The Cargo.lock churn is exactly what the toml change implies.

## What I learned

Hidden subcommands accumulate. Codex has at least three (`ResponsesApiProxy`,
`StdioToUds`, the now-deleted `Responses`) and the right governance question
is whether each one belongs as a subcommand or as a flag on a sibling. The
"one-shot vs daemon" distinction here is the kind of split that should
collapse into a `--once` flag the moment one of the two is provably dead —
which is what this PR is acting on.
