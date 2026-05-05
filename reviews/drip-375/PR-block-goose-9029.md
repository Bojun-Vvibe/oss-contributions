# block/goose #9029 — agents: add CLAUDE.mds to mirror AGENTS.mds

- **PR:** block/goose#9029
- **Head SHA:** `655e7f4296015c5b9fe9870bf28292acccd9f063`
- **Files:** 5 changed (+5 / -0) — five new files, each containing a single line

## What changed

- Five new `CLAUDE.md` files added at the same paths where `AGENTS.md` already exists: `CLAUDE.md`, `documentation/CLAUDE.md`, `ui/goose2/CLAUDE.md`, `ui/goose2/src/shared/ui/CLAUDE.md`, `ui/text/CLAUDE.md`. Each contains exactly one line: `@AGENTS.md` — a Claude-Code-style include directive that tells the assistant to load the sibling `AGENTS.md` file as instructions instead of duplicating the content.
- All five files share the same blob hash (`43c994c2d361`) confirming they're byte-identical, which keeps the maintenance contract obvious: keep editing `AGENTS.md`, `CLAUDE.md` is just a forwarder.

## Risks / notes

- This is purely a discoverability shim for a different agent client. There's zero behavioural risk to the goose codebase itself — the files are read only by external AI tooling and ignored by anything checking out the repo for build purposes.
- The `@AGENTS.md` include syntax is a Claude Code convention. Other agent clients that look for `CLAUDE.md` (e.g. some forks) but don't understand `@`-imports will see "@AGENTS.md" as literal user-facing prose, which is harmless but ugly. Worth noting for future agents that may parse it differently.
- Coverage check: the goose repo has at least one other `AGENTS.md` not mirrored here — `~/Desktop/...` paths excluded, but `goosed/AGENTS.md`, `crates/.../AGENTS.md` etc. should be checked. If the intent is "mirror everywhere", a `find . -name AGENTS.md` audit should be in the PR description; if the intent is "mirror at root and main UI surfaces", that limit should be documented.
- A pre-commit hook or simple CI check that flags any new `AGENTS.md` without a matching `CLAUDE.md` (or vice versa) would prevent drift. Not required, but the pattern is begging for it.

## Verdict

**merge-as-is** — five-line, blob-identical shim files with documented intent. The "mirror everywhere?" follow-up question is worth answering in the PR description but doesn't gate the merge.
