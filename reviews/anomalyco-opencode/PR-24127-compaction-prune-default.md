# PR-24127 — fix: enable compaction prune by default

[sst/opencode#24127](https://github.com/sst/opencode/pull/24127)

## Context

One-line behavior flip in `packages/opencode/src/session/compaction.ts`:
the `prune` Effect early-returned when `cfg.compaction?.prune` was falsy
(`!cfg.compaction?.prune`). With the default config object having no
`compaction` block, `cfg.compaction?.prune` was `undefined` — i.e. prune
was effectively *opt-in*. The fix changes the guard to
`cfg.compaction?.prune === false`, so only an explicit `false` disables
pruning. The accompanying compaction test was split into two
fixtures: one asserting "prunes by default" and one asserting "does not
prune when `compaction: { prune: false }`", with a shared
`createPrunableToolSession(dir)` helper to keep both branches honest.

## Why it matters

The whole point of the prune pass is to erase old completed tool output
so context-window pressure doesn't blow up long sessions. Shipping it
opt-in meant most users were silently running with the un-pruned
behavior even though the codepath existed and the team thought it was on.

## Strengths

- Smallest possible behavior change: one operator flip, default-on
  semantics, no surface change to the config schema.
- Test split is the right shape — extracting the prunable-session setup
  into a helper means the "default on" and "explicit off" assertions
  share the *exact* same fixture, so the only thing under test is the
  config interpretation. That's the kind of paired-test layout that
  catches regressions where someone tightens the guard back to `!cfg…`.
- Asserts on `part.state.time.compacted` being a number in the on-path
  and `undefined` in the off-path — checks the observable side effect,
  not just "the function returned".

## Concerns / risks

- This is a silent behavior change for anyone who *intended* to be
  un-pruned by relying on the previous default. There's no schema
  default annotation in the diff and no migration / changelog text in
  the patch — operators with long-running sessions where they want full
  tool output preserved (e.g. for auditing, replay, post-hoc inspection)
  will start seeing tool outputs erased without any local config change.
- The new default depends on whatever heuristic `prune` uses for "old"
  vs "fresh" tool output. The diff doesn't touch that heuristic, but
  flipping the default makes it the production codepath for everyone —
  any latent bug there now hits 100% of users.
- `cfg.compaction?.prune === false` requires the user to pass *exactly*
  the boolean `false` to opt out. If a user writes `prune: 0` or
  `prune: "false"` in JSON/TOML thinking they're disabling it, they'll
  silently get the on-path. The schema layer presumably coerces, but
  this is worth a sanity test.
- No assertion that the new default doesn't fire on sessions that
  *don't* have prunable content — i.e. the early-return should still
  short-circuit cheaply for short sessions. That's not covered.

## Suggested follow-ups

- Add a CHANGELOG / release-notes line: "compaction.prune is now on by
  default; set `compaction.prune: false` to restore previous behavior."
- Add a third test variant: explicit `compaction: { prune: true }`
  behaves identically to the default-on path (regression guard against
  someone reintroducing tri-state logic later).
- Consider promoting `compaction.prune` from optional to required-with-
  default in the config schema so the default surfaces in
  `opencode config show` / docs, instead of being an implicit boolean
  buried in code.
