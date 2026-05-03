# Review: block/goose #8945 — remove artifacts dir handling

- **Repo**: block/goose
- **PR**: #8945
- **Head SHA**: `d6c0e2dd8df2d7bf4906ad0343ebc468d288d3ce`
- **Author**: jamadeo

## What it does

Removes the per-project `artifacts/` directory concept from the goose2
desktop UI: drops the Tauri capability allowance for `$HOME/.goose/artifacts/**`,
removes the `project_artifacts_dir()` helper and the `artifacts_dir`
field on the serialized `ProjectInfo`, and updates `path_resolver` tests
to use neutral (`Documents`, `src`) path components instead of
`artifacts`.

## Diff notes

- `ui/goose2/src-tauri/capabilities/default.json:22-24` — removes
  `{ "path": "$HOME/.goose/artifacts/**" }` from the
  `opener:allow-open-path` list. The broader `$HOME/.goose/**` allowance
  (line 19) still covers the path if anything legacy still writes there,
  so this is a reduction in *explicitness* rather than access. Worth
  asking the author whether the intent is to also block `$HOME/.goose/artifacts/`
  — if so, the `$HOME/.goose/**` wildcard above defeats that.
- `ui/goose2/src-tauri/gen/schemas/capabilities.json` — generated
  artifact regenerated to match. Good (it would have caused a CI drift
  otherwise).
- `ui/goose2/src-tauri/src/commands/projects.rs` — drops `use Path`,
  removes `project_artifacts_dir()`, and removes `artifacts_dir` from
  the `ProjectInfo` struct returned to the frontend. **This is a
  breaking change for the frontend type**. Any TS consumer reading
  `projectInfo.artifactsDir` will now get `undefined`. Need to
  cross-check the React/UI side: are the artifacts-dir consumers also
  removed in this PR, or is this PR landing a backend that no longer
  feeds a still-live UI?
- `ui/goose2/src-tauri/src/commands/path_resolver.rs:tests` — neutral
  path swaps (`artifacts` → `src`/`Documents`). Pure test cleanup,
  no behavior change.

## Concerns

1. **Frontend coupling**: removing `artifacts_dir` from `ProjectInfo`
   without seeing the corresponding TS type / consumer change is a red
   flag. Either (a) the frontend already stopped reading it (in a prior
   PR or earlier commit on this branch), or (b) this PR ships a broken
   contract. The PR should call out which.
2. **Migration story**: Existing user installs likely have
   `$HOME/.goose/artifacts/` populated. Removing UI surface area for
   it without a one-time migration / cleanup leaves orphan files on
   disk. Nice-to-have: a CHANGELOG entry pointing users at the
   directory.
3. **Bundle of concerns**: This PR is doing capability removal +
   field removal + helper removal + test rename, all in one. Each one
   is small, but together it warrants a slightly fuller PR description
   than "remove artifacts dir handling".

Cleanup PR; correct in shape. Block on confirming the frontend half is
either gone or going.

## Verdict

request-changes
