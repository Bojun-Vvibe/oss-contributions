# block/goose PR #8955 — chore(deps): bump dependabot/fetch-metadata from 2.3.0 to 3.1.0

- **PR:** https://github.com/block/goose/pull/8955
- **Author:** app/dependabot
- **Head SHA:** `65db8cd6` (full: `65db8cd6275a9f1f1ec7452719f948b66c42f8db`)
- **State:** OPEN
- **Files touched:**
  - `.github/workflows/dependabot-auto-merge.yml` (+1 / -1)

## Verdict

**merge-after-nits**

## Specific refs

- `.github/workflows/dependabot-auto-merge.yml:17` — single-line change, action SHA bumped from `d7267f607e9d3fb96fc2fbe83e0af444713e90b7` (v2.3.0) to `25dd0e34f4fe68f24cc83900b1fe3fe149efef98` (v3.1.0). SHA pinning is preserved — good. Verified the new SHA resolves to the v3.1.0 release tag on the upstream repo.
- The release notes (per PR body) cite "Add permissions to all workflows" as the headline v3.0.0 change, plus ecosystem expansions. v3.x bumps the action's minimum runtime; need to confirm the `ubuntu-latest` runner this workflow uses (implicit in `dependabot-auto-merge.yml`) still satisfies it. v3.1.0 release notes don't flag a runner-version requirement bump, so this should be safe.

## Rationale

Trivial dependency bump in an auto-merge plumbing workflow. Single line, SHA-pinned, upstream is a well-known GitHub-published action.

The only reason this isn't `merge-as-is` is that `dependabot-auto-merge.yml` is a *trust-sensitive* workflow — it's the thing that auto-merges other dependency bumps. A regression here silently widens the auto-merge surface. Two cheap pre-merge checks would resolve that:

1. Confirm the v2 → v3 metadata output schema is identical for the fields this workflow consumes (the workflow likely reads `steps.metadata.outputs.update-type`; v3.x has not changed that output, but worth verifying against `with:` inputs in the truncated diff).
2. Run the workflow once in a dry-run / non-merging mode against an open dependabot PR to confirm the gate logic still triggers correctly.

If those check out, merge.
