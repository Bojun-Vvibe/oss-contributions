# BerriAI/litellm PR #27220 — chore: upgrade click dependency to 8.3.3

- URL: https://github.com/BerriAI/litellm/pull/27220
- Head SHA: `520116f5cac3e59d4ccc09d207d77b7560f455fc`
- Size: +1 / -1

## Summary

One-line bump in `pyproject.toml` of the exact pin `click==8.1.8` →
`click==8.3.3`. PR body claims `Fixes: #27174` (which would need to be
verified by a maintainer; the body otherwise has every checklist item
unchecked).

## Specific findings

- `pyproject.toml:20` — single string literal swap from `"click==8.1.8"` to
  `"click==8.3.3"`. No range widening, no migration to `>=,<`, just an
  exact-pin bump. Consistent with the existing `==`-pin style across this
  dependencies block (the floor-CI / ranges work in #27241 hasn't landed
  yet).
- Click 8.1.x → 8.3.x crosses two minor versions. Notable surface changes
  to consider:
  - 8.2.0 dropped Python 3.7 support (litellm already requires 3.10+, so
    no impact).
  - 8.2.0 deprecated `click.BaseCommand` — direct subclasses now warn.
    `rg 'BaseCommand' litellm/` should be clean.
  - 8.2.x removed several long-deprecated aliases (`click.utils.safecall`,
    etc.); a `rg 'from click' litellm/` audit should confirm.
  - 8.3.x added the `Click 8.x release notes` deprecation of
    `Context.invoked_subcommand` semantics on chained groups — relevant
    only if litellm CLI uses `chain=True` groups.

## Notes

- Zero test edits. For a runtime-dep version bump, a smoke test exercising
  the litellm CLI entry point (`litellm --help`, `litellm --version`)
  would catch the most obvious breakage from removed/renamed click APIs.
  The repo's existing CLI smoke tests (if any) presumably exercise this
  path; explicit confirmation in the PR body would strengthen it.
- PR description does not enumerate *why* 8.3.3 specifically (CVE? bug
  fix? compat with another dep?). The "Fixes #27174" link is the only
  context. A one-sentence rationale ("8.1.8 had X, 8.3.3 fixes Y") would
  help future bisects.
- All checklist boxes unchecked — including "test added", "PR scope
  isolated", and the Greptile review step. For a one-line dep bump the
  scope is trivially isolated, but the absence of any green-CI evidence
  in the PR body means a maintainer must verify CI on the PR head before
  merge.
- No CHANGELOG entry. Dep-bump PRs in this repo historically don't add
  one, so this matches existing convention.

## Verdict

`merge-after-nits`
