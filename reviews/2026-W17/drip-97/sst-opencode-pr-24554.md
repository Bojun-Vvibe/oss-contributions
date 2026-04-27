# sst/opencode#24554: refactor: remove module barrels

- **Author:** thdxr (Dax)
- **HEAD SHA:** cff0d9e2
- **Verdict:** merge-after-nits

## Summary

A 727/-677 sweep across 300+ files removing module barrel `index.ts`
files and rewriting imports to target specific modules directly. The
stated motivation is real and substantive: barrel files were causing
cyclic-init failures (`MessageV2.Assistant` initialization
specifically), and module barrels in TypeScript-with-bundler setups
routinely degrade tree-shaking, blow up cold-start time, and convert
otherwise-local edits into whole-package rebuilds. The PR is from a
maintainer (Dax), the verification is `bun typecheck` plus a single
`bun run src/temporary.ts --help` smoke, and the diff shape (1-3
lines per file, hundreds of files) is exactly what a barrel removal
should look like — almost no logic touched, mostly import-target
rewrites.

The interesting concentrated edits live where *some* re-export is
still wanted: `src/effect/runner.ts` (+2/-0),
`src/cli/cmd/tui/util/clipboard.ts` (+2/-0),
`src/cli/cmd/tui/util/selection.ts` (+2/-0), and
`src/cli/cmd/tui/util/sound.ts` (+3/-1) all gained self-reexports
where namespace imports were still useful — which is the right
approach for the common pattern `import { Util } from "./util"` on a
single module rather than a directory.

## Specific feedback

- `packages/opencode/src/effect/index.ts` (-5) and
  `packages/opencode/src/config/index.ts` (-16) and
  `packages/opencode/src/lsp/index.ts` (-3) and
  `packages/opencode/src/cli/cmd/tui/util/index.ts` (-4) and
  `packages/opencode/src/cli/cmd/tui/plugin/index.ts` (-3): the
  deleted barrels are exactly the cycle-implicated ones. Worth a
  one-line PR-body callout naming which barrel was the actual
  culprit for the `MessageV2.Assistant` init failure so the next
  person hitting "why did we delete these?" has the receipt.
- `packages/opencode/src/effect/app-runtime.ts` (+12/-12) is the
  largest single-file rewrite — it's pure import-fanout and looks
  mechanical, but it's the file most likely to have a subtle
  side-effect-ordering regression because Effect runtime boot order
  matters. Worth a quick cold-boot smoke beyond `--help`.
- `packages/opencode/src/control-plane/workspace.ts` (+8/-5) and
  `packages/opencode/src/lsp/lsp.ts` (+6/-4) net-add lines, which
  for a "remove barrels" PR usually means an inline import that
  used to be a named re-export got expanded to a multi-line `import
  { ... } from "..."`. Fine, but worth eyeballing the LSP one
  because LSP touches both worker boot and request paths.
- The verification `bun run src/temporary.ts --help` is a thin
  smoke. Recommend a `bun test` run if there's any test surface
  here, or at minimum a manual session-prompt round-trip to confirm
  the original `MessageV2.Assistant` cycle is actually gone.
- No new self-reexport for the `bus`, `command`, `format`, `mcp`,
  or `effect` packages even though their `index.ts` files still
  exist with content (`bus/index.ts +3/-3`, `format/index.ts
  +3/-3`, `mcp/index.ts +4/-4`). Confirm those are intentional
  keeps (real public-surface modules) vs. survivors that should
  also be removed in a follow-up.

## Risks / questions

- Cycle-detection regression: this PR is a structural change, not a
  feature, so the right post-merge guard is a CI lint (e.g.,
  `madge --circular` or equivalent) so the next barrel-induced
  cycle gets caught at PR-time rather than at boot. Worth filing as
  a follow-up.
- The diff exceeded GitHub's 300-file diff limit so individual-file
  review via the PR UI is degraded — reviewers will need to clone
  locally or work file-by-file via the API. Not blocking, but
  contributes to the merge-after-nits posture.
- No `.changeset/` or version-bump entry visible in the file list;
  if this package publishes, confirm whether this counts as a patch
  bump (no public API change) or whether the self-reexport additions
  count as a new public surface.
