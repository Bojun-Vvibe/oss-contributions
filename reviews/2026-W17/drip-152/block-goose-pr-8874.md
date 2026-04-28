# block/goose#8874 — docs: hide Windows CUDA download links until release

- **Repo:** [block/goose](https://github.com/block/goose)
- **PR:** [#8874](https://github.com/block/goose/pull/8874)
- **Head SHA:** `fcd3fbdcb197592a645028756529e42899a00312`
- **Size:** +0 / -32 across 2 files (`installation.md`, `WindowsDesktopInstallButtons.js`)
- **State:** MERGED

## Context

The Windows CUDA build artifacts aren't shipped in the current release
yet — the previous documentation update added install instructions and
download buttons for them ahead of the release pipeline actually
producing the binaries. So users following the docs would either get
404s on the download URLs or run install scripts that referenced
non-existent release assets. Pure deletion PR walking back the
premature documentation.

## Design analysis

Two surfaces affected, both pure deletion:

1. **Documentation prose deletion** at
   `documentation/docs/getting-started/installation.md`. Removes:
   - The `:::info Windows variants` callout box (`:144-146`)
     describing the standard-vs-CUDA distinction
   - The "Use the standard build... Use the CUDA variant if you have
     an NVIDIA GPU" sentence (`:159`)
   - Two CUDA-variant `curl | GOOSE_WINDOWS_VARIANT=cuda bash`
     commands (`:165-169` and `:177-181`)
   - The PowerShell CUDA installation block (`:196-202`)

   What's preserved: every standard-Windows install path (`curl |
   bash`, `CONFIGURE=false`, PowerShell standard) is left intact.
   Right scope — the deletion is surgical to CUDA-specific lines,
   not broad to the Windows section.

2. **Component button deletion** at
   `documentation/src/components/WindowsDesktopInstallButtons.js:14-19`.
   Removes the `<Link>` to
   `https://github.com/aaif-goose/goose/releases/download/stable/Goose-win32-x64-cuda.zip`
   from the button group. Leaves the standard `Windows` button
   intact. Right scope.

The PR body's one-line description ("CUDA downloads won't be available
until the next release is made, so hide the links for now") correctly
frames this as temporary — implying the lines should come back when
the release pipeline produces the artifacts.

## Risks / nits

1. **Deletion-instead-of-comment is a temporary-fix anti-pattern.**
   The PR body explicitly says "until the next release is made" —
   meaning these lines need to come back. Pure deletion forces the
   re-add to be either (a) a fresh hand-write of the same content
   (drift risk: the re-added prose may differ subtly from the
   original) or (b) a `git revert` of this commit (works but
   couples the future re-enable PR's diff to this PR's exact
   commit hash). Better shape: HTML/MDX comments
   (`{/* TODO: restore after CUDA release ships, see PR #8874 */}`)
   wrapping the blocks, leaving them in source but invisible. That
   way the re-enable is a one-line uncomment.

2. **No corresponding cleanup of `GOOSE_WINDOWS_VARIANT=cuda`
   handling in the install scripts.** The install scripts
   themselves (`download_cli.sh`, `download_cli.ps1`) presumably
   still accept the `GOOSE_WINDOWS_VARIANT=cuda` env var since
   the prior PR added the docs alongside script support. With the
   docs removed, the script still has dead-by-documentation code
   paths. Either:
   - Document in the PR description that the script paths are
     still functional but undocumented (advanced-user escape hatch
     until the release ships), or
   - Add a runtime warning in the script when
     `GOOSE_WINDOWS_VARIANT=cuda` is set saying "CUDA variant not
     yet released; falling back to standard."

   Without one of those, advanced users who copy-pasted the CUDA
   curl command from the *previous* version of the docs (cached in
   browsers, in their shell history, in their internal wikis) will
   still hit the broken release-download URL.

3. **Tracking-issue link missing from PR body.** "Until the next
   release is made" is a future commitment but there's no linked
   issue or release-blocker tag. Worth opening a tracking issue
   ("Restore Windows CUDA documentation after release X.Y.Z ships")
   and linking from this PR so the re-enable doesn't get lost when
   the release pipeline catches up.

4. **The `aaif-goose/goose` repo URL in the deleted button is
   notable.** The button URL pointed at `aaif-goose/goose` not
   `block/goose` — meaning the CUDA build was being shipped from
   a fork or a release-channel mirror separate from the main repo.
   This isn't relevant to the deletion PR but worth noting because
   when the docs come back, the URL convention may change.

## Verdict

**merge-as-is.** The deletion is surgical, the scope is correctly
limited to CUDA-only lines while preserving every standard-Windows
install path, and the change is reversible. The PR body's framing as
temporary is honest. Nits 1 (HTML-comment instead of pure-delete)
and 3 (tracking issue) are real but not blocking — both can be
addressed in follow-up. Nit 2 (script-side cleanup) is a separate
concern that warrants its own issue.

## What I learned

- "Premature documentation" is a real failure mode when docs are
  written ahead of the release pipeline that produces the
  artifacts. The right cadence is doc-changes-after-release-asset-
  exists, or doc-changes-gated-on-release-flag.
- Pure deletion of lines that need to come back later is
  weaker than HTML/MDX-commenting them out. The deletion forces
  either re-write (drift risk) or git-revert (coupling). Comments
  preserve the source-of-truth in version control while hiding
  from rendered output, and the re-enable is a one-line change.
- Documentation buttons that link to release-download URLs are
  effectively a contract — if the docs say the URL exists, users
  expect the URL to work. The "easiest documentation" (a button
  with a hardcoded URL) is also the one that 404s most visibly
  when the upstream artifact disappears or moves.
- The `aaif-goose/goose` namespace appearing in a `block/goose` repo
  hints at a fork/mirror release pipeline. Documentation that
  hardcodes URLs to a different namespace than the source-repo's
  own organization is a coupling worth surfacing in code review —
  the docs may keep working, may break, or may silently swap to a
  different release pipeline depending on who controls the URL.
