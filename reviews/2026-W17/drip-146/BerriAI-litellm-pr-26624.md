# BerriAI/litellm #26624 — Add note that docs live in separate litellm-docs repo

- PR: https://github.com/BerriAI/litellm/pull/26624
- Head SHA: `7d2f029acb90fcea7ec30073b40c74ccb63a1ad9`
- Author: krrish-berri-2
- Files touched: `.github/pull_request_template.md`, `CONTRIBUTING.md`

## Observations

- `.github/pull_request_template.md:46-48` — adds two HTML comments and inlines the link `📖 Documentation (submit doc PRs to [litellm-docs](https://github.com/BerriAI/litellm-docs))` into the PR-type checklist. The comment block above is invisible in rendered preview but visible to anyone editing the template, which is the right place to land the warning.
- `CONTRIBUTING.md:245` — the bullet `Update documentation: If you change APIs, update docs` becomes `... update docs in the [litellm-docs](https://github.com/BerriAI/litellm-docs) repo (see [Documentation](#documentation) below)`. Adds a forward link.
- `CONTRIBUTING.md:315-319` — new `## Documentation` section explicitly says "Do not create or edit documentation files in this repository." Clear, unambiguous.
- `CONTRIBUTING.md:345` — the "Documentation improvements" idea entry under contribution opportunities now points at the docs repo too. Consistent.
- No risk: pure markdown edits, no code paths affected.

## Verdict: `merge-as-is`

**Rationale:** Cleanly redirects contributors to the right repo for docs work, which is exactly the kind of friction this kind of PR template warning prevents. Internally consistent across all four edits.
