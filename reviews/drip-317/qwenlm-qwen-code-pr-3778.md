# Review: QwenLM/qwen-code #3778 — feat(desktop): Add desktop app package with Qwen ACP SDK integration

- PR: https://github.com/QwenLM/qwen-code/pull/3778
- Head SHA: `179df182a3e2a8ae1dc0c7a550b857b0cbd242df`
- Author: DragonnZhang
- Size: +325026 / -3 (over 300 files; gh diff endpoint returns
  `too_large` so the review here is structural, based on the file
  manifest and PR description)

## Summary

Imports a fork of an existing third-party desktop app (described in
the PR body as "Craft/Claude Desktop fork") under
`packages/desktop/` and wires the Qwen ACP SDK into it for skill
discovery, session management, and context usage tracking. Also
edits root `package.json` and `package-lock.json` (delta of +2/-1 and
+2/-2 respectively) to *exclude* the new package from the root npm
workspace so the desktop app's dependency graph is isolated. A small
undici type-usage fix is mentioned in the body.

## Specific citations

- `package.json` (+2 / -1) — workspace exclusion. The PR body
  explicitly says the desktop package is excluded from the root
  workspace; this 3-line delta is presumably an entry like
  `"!packages/desktop"` in the `workspaces` array. Confirm the actual
  field; npm `workspaces` supports negative globs but pnpm and yarn
  classic do not, so portability matters here.
- `packages/desktop/.dockerignore`, `packages/desktop/Dockerfile.server`,
  `packages/desktop/.github/workflows/{validate,validate-server}.yml`
  — the package ships its own Docker build and its own GH Actions
  workflows. With nested `.github/workflows/` directories under a
  subpackage, GitHub Actions does NOT auto-discover them (only the
  repo-root `.github/workflows/` is scanned). So these files are
  inert at the upstream repo level — they only matter if the package
  is re-imported into a standalone repo. That should be made
  explicit, or these files should be moved to the upstream
  `.github/workflows/` with appropriate path filters, or deleted to
  reduce confusion.
- `packages/desktop/LICENSE` (+191), `packages/desktop/NOTICE` (+8),
  `packages/desktop/CODE_OF_CONDUCT.md`, `packages/desktop/SECURITY.md`,
  `packages/desktop/TRADEMARK.md` — third-party licence and notice
  files. These need to be reconciled with the upstream qwen-code
  licence. If the imported fork is Apache-2.0 (likely, given
  `NOTICE`), the combined repo licence terms have to permit the
  combination. A `LICENSE.md` audit by the qwen-code maintainers is
  a precondition to merge.
- `packages/desktop/apps/cli/`, `packages/desktop/apps/electron/` —
  two separate apps (a CLI and an Electron shell). +325k lines is
  enormous for one PR; these are independently scopable and should
  arguably be two PRs (or split via a series). Reviewing 325k lines
  of imported third-party code at the diff level is not practical;
  the qwen-code maintainers will need a different review strategy
  (audit the import script + spot-check + run the test suites).

## Verdict

**needs-discussion**

## Rationale

This is a vendor/import PR rather than a normal feature PR, and it
should be evaluated as such. The mechanical questions a reviewer
needs answered before merging:

1. **Provenance**: which exact upstream commit/tag of which exact
   repo was imported? The PR body says "Craft/Claude Desktop fork"
   but doesn't pin a commit. Without a pin, future "sync from
   upstream" PRs are not reproducible and security CVEs against the
   original repo can't be tracked. A `packages/desktop/UPSTREAM` file
   recording `<repo-url>@<commit-sha>` is table stakes.
2. **Licence**: see citation above. `LICENSE`, `NOTICE`, and
   `TRADEMARK.md` co-exist with the qwen-code root licence. The
   maintainers (not me) need to confirm this is a permitted
   combination and that attribution is adequate.
3. **Workspace isolation strategy**: excluding from the root
   workspace means the desktop package has its own lockfile and
   dependency graph. That's a defensible choice for a vendored fork,
   but it doubles the dependabot/security-scan surface and means
   `npm ci` at the root never touches it. Document the
   build/install/test commands (presumably
   `cd packages/desktop && npm ci && npm test`) in the root README
   or a `packages/desktop/README.md` (which exists at +68 lines —
   confirm it covers this).
4. **Dead workflows**: the nested `.github/workflows/` files do
   nothing at the upstream level. Either delete them or call out the
   plan for them.
5. **Diff size and review feasibility**: 325k lines from a single
   author in a single PR cannot be reviewed line-by-line. The qwen-
   code maintainers should require either a split (CLI vs Electron),
   a vendored-import script that can re-derive the diff
   deterministically, or both.

The Qwen ACP SDK integration described in the body is the *feature*
content of this PR but it's invisible against the import bulk. Once
the import is split out, the actual Qwen-specific glue code becomes
reviewable on its own merits.

I scanned the file manifest and found nothing that violates this
repo's own naming/security policies (no banned strings in the
manifest or PR body). That is necessary but not sufficient — full
content review of 100+ vendored files is out of scope here.

## Discussion items

1. Pin the upstream provenance with a commit SHA in a checked-in
   `UPSTREAM` file.
2. Maintainer licence audit.
3. Split the Electron and CLI apps into two PRs, or land the import
   first via a vendored-tarball PR and the Qwen integration as a
   follow-up.
4. Decide the fate of nested `.github/workflows/` (delete vs hoist).
5. Document the build/test workflow for a workspace-excluded
   package in the root contributor docs.
