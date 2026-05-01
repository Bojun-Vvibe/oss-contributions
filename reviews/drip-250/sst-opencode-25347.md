# sst/opencode #25347 — chore: drop placeholder random/hello scripts from root package.json

- URL: https://github.com/sst/opencode/pull/25347
- Head SHA: `e3ce419a9d30f155e1753a862d8ed2e1f91931c9`
- Files: `package.json` (+0/-2)

## Context

Drops two scripts from the root `package.json`:
- `"random": "echo 'Random script'"`
- `"hello": "echo 'Hello World!'"`

Author verified via `grep -rn "bun run random\|npm run random\|bun run hello\|npm run hello" packages script .github` — zero callers in workspace, in shell scripts under `script/`, or in CI workflows under `.github/`. Two-line surgical delete.

## Risks

- Two minor untested vectors not covered by the grep:
  1. **Workspace package scripts** — the grep walks `packages/`, `script/`, `.github/` but a `bun run random` could in principle live in a sub-package's `scripts.*` body (a workspace-resolved `bun run` would still hit the root). Quick `grep -rn '"random"\|"hello"' packages/*/package.json` would tighten the proof.
  2. **External docs / contributor onboarding READMEs** — if any README snippet says `bun run hello` to verify their install, deleting it is a tiny breaking change. Worth a 30-second pass through `README.md` and `CONTRIBUTING.md`.
- Both are nit-level. Net `-2` on a script set that everyone else has to scroll past in `bun run` tab completion.

## Verdict

`merge-as-is` — placeholder scripts that don't even pretend to do useful work, no callers in any of the obvious places. Tab-completion noise reduction is its own justification. The two grep extensions above would tighten the audit but the failure mode if missed is "someone's local muscle-memory `bun run hello` errors with `script not found`" which is fine.
