# charmbracelet/crush #2801 — chore: fixed typo in hooks README

- Head SHA: `de9d901ef6f9440effa98d8feaff6a3cc51bdcb4`
- Author: ardevd
- Diff: +1 / −1 across 1 file (`docs/hooks/README.md`)

## Verdict: `merge-as-is`

## What it does

One-character grammar fix in `docs/hooks/README.md:15`:

```diff
-- Hooks just shell commands
+- Hooks are just shell commands
```

## Why this is fine to land

1. **The fix is correct.** Pre-fix: "Hooks just shell commands" reads as a sentence fragment (missing copula). Post-fix: "Hooks are just shell commands" is grammatical and matches the parallel structure of the surrounding bullets ("Hooks can be written...", "Hooks are Claude Code-compatible", "Crush ships with..."). Two of the four sibling bullets already start with "Hooks are/can/ship", so the typo bullet was the one that broke the pattern.
2. **Single-line, no risk surface.** Pure documentation, no behavioral change, no link breakage, no formatting churn around the change.
3. **PR template box checked** (CONTRIBUTING.md acknowledged in the PR body).

## Nits

None. This is the canonical "merge-as-is" change.

## Optional follow-up (not for this PR)

While reading the surrounding bullets I noticed the next line reads:
> Crush ships with a builtin `crush-hook` skill write, edit, and configure

That looks like it's missing a `to` (probably should be `crush-hook` skill *to* write, edit, and configure). A separate one-line PR would close that out. Not the responsibility of this contributor.

## Citations

- `docs/hooks/README.md:15` — the one-character fix
