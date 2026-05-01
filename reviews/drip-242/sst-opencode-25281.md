# Review: sst/opencode #25281 — fix(tui): long tips truncated in TUI home view

- URL: https://github.com/sst/opencode/pull/25281
- Head SHA: `d47fd19dabd3599e7001aacd50012c6b57682dba`
- Base: `dev`
- Size: 2 files, +19/-6
- Closes: #25279

## Summary of intended change

The TUI home view's `Tips` carousel renders `"● Tip " + TIPS[idx]` inside a
parent component pinned to a 75-column box. Several entries in `TIPS` (when
the `{highlight}…{/highlight}` markup is stripped) exceeded 69 visible
characters, so the parent's word-wrap dropped the trailing word, producing
truncated tips like `"...containerized"` (issue #25279).

This PR does two things:
1. Exports `TIPS` from
   `packages/opencode/src/cli/cmd/tui/feature-plugins/home/tips-view.tsx:53`
   and adds a comment at `:54-56` documenting the 69-char visible-length
   budget.
2. Rewrites four offending tips at lines `:63`, `:89`, `:131`, `:149-151` to
   bring stripped length under 69.
3. Adds `packages/opencode/test/cli/tui/tips.test.ts` (new file, +10) that
   filters `TIPS` for any whose stripped length exceeds 69 and asserts the
   filtered array is empty — the perfect shape: future tip additions will
   fail this test at PR time.

## Review

### Test correctness

The regex at `tips.test.ts:8`:
```ts
TIPS.filter((tip) => tip.replace(/\{\/?highlight\}/g, "").length > 69)
```
strips both `{highlight}` and `{/highlight}` markers (the `\/?` makes the
trailing slash optional). That matches exactly the markers used in `TIPS`,
so the visible-length math should agree with what the renderer measures.
One edge case the test does **not** cover: any future `{highlight}` token
inside another tag, or a tip that uses a different markup syntax (e.g.
`<b>...</b>`). Probably fine since the codebase only uses `{highlight}`,
but worth a one-line note in the test that it's coupled to the renderer's
markup vocabulary.

### Rewrites — visible-length spot check

Each rewrite must come in under 69 visible chars after stripping markup.
Three are clear wins (`Run /share to publish your conversation at opencode.ai`
≈ 53 chars; `Use opencode.json for server config, tui.json for TUI config`
≈ 60 chars; `Use {env:VAR_NAME} to reference env variables in config` ≈ 55
chars). The fourth at `:151`
`Use docker run --rm ghcr.io/anomalyco/opencode for containerized use` is
the borderline one — counting visible chars (without the `{highlight}` tags)
gives 66, comfortably under 69. The test will pin this for future edits.

### One semantic regression risk

The pre-PR text at `:151` was
`Run docker run -it --rm ghcr.io/anomalyco/opencode for containerized use`.
The PR drops `-it` (keeps `--rm`), changing the suggested command from
"interactive TTY" to "non-interactive". For an opencode TUI demo, `-it` is
load-bearing (without `-t` you don't get a usable terminal, and without
`-i` you can't type into the prompt). The old command was the right one;
the new one will hand new users a broken first-run. **Recommend** putting
`-it` back and trimming somewhere else (e.g. `Use docker run -it --rm
ghcr.io/anomalyco/opencode in containers` ≈ 60 chars).

### Symmetry of TIPS export

Marking `TIPS` `export`-able at `:53` widens the module's public surface
just to support the test. That's fine, but `TIPS` is now importable from
anywhere in the codebase. If you'd rather not commit to that, factor the
length-checker into a helper exported from a `tips-view.test-helpers.ts` and
re-import the constant via that helper. Low priority.

## Verdict

**merge-after-nits**

Wants:
- Restore `-it` in the docker tip at `tips-view.tsx:151` — the rewritten
  command produces a broken container experience for users who copy it.
- Optional: one-line test comment that the regex is coupled to the
  `{highlight}` markup vocabulary.
- Optional: future-proof by also asserting `TIPS.length > 0` so an
  accidental empty-array regression is caught.

Test infrastructure is exactly the right shape for this class of issue.
