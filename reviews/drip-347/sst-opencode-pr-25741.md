# sst/opencode#25741 — feat: Package OpenCode as a Snap

- **Head SHA**: `68a71c734ae8e67635c0e8118aafea4b44da12ca`
- **Verdict**: `request-changes`

## Summary

Adds a single new file `snap/snapcraft.yaml` (+172/-0) defining two parts (`opencode` and `opencode-desktop`) that vendor Bun, Node 22, and a desktop Electron build into a `core24` classic-confinement snap. Closes #25736.

## Findings

- `snap/snapcraft.yaml:24` declares `confinement: classic` with `grade: stable`. The Snap Store will reject a `classic` confinement publish without a documented review request — this needs justification in the PR body (the CLI shells out to user-installed compilers/LSPs, which is a defensible reason, but it should be stated explicitly so reviewers don't get blocked).
- `snap/snapcraft.yaml:50-58` and `:128-136` both `curl | bash` the Node and Bun installers inside `override-build`. Snap builds run sandboxed but pinning the installer revisions (or fetching versioned tarballs) avoids silent breakage when nodesource/bun ship incompatible installers. At minimum, pin the Node major in a comment near `setup_22.x`.
- `snap/snapcraft.yaml:62-64` runs `bun install` at workspace root then `cd packages/opencode && bun run script/build.ts --single`. The desktop part at `:144-150` runs `bun install` again — fine, but the `after: [opencode]` ordering means the second `bun install` re-resolves the entire workspace. Consider `--frozen-lockfile` to make the build reproducible.
- `:163-170` writes the `.desktop` file via `printf` rather than checking in a real desktop entry. Works, but a tracked file under `snap/local/` would be easier to audit and lint with `desktop-file-validate`.
- No `apps.opencode-cli.plugs:` declared. Under classic confinement plugs are largely advisory, but absence means future strict-mode conversion will need a full audit. Not a blocker.

## Recommendation

Hold for: (1) pinned installer versions or an explicit "we accept upstream drift" note, (2) classic-confinement justification in the PR description, (3) `--frozen-lockfile` on both `bun install` invocations. The mechanics look correct otherwise; build artifacts at `:78-83` and `:158-161` map cleanly to the existing deb layout.
