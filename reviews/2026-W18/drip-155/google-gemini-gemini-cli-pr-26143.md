# PR #26143 — refactor(acp): modularize monolithic acpClient into specialized files

- **Repo:** google-gemini/gemini-cli
- **Link:** https://github.com/google-gemini/gemini-cli/pull/26143
- **Author:** sripasg (Sri Pasumarthi)
- **State:** OPEN
- **Head SHA:** `19b1f37c99702227eaaf513e651093475a77f855`
- **Files:** 16 changed (+2249/-3163)

## Context

`packages/cli/src/acp/acpClient.ts` and its test file had grown to be a single monolithic surface owning: stdio transport, RPC dispatch, multi-session lifecycle, per-turn session state, and shared utilities. The test file alone was 2294 lines (deleted in this PR — `acpClient.test.ts` -1/+0 with -2294 deletions in the file diff). Phase 1 of the ACP refactor splits these into discrete modules.

## What changed

Five new source modules:
- `acpStdioTransport.ts` (35 lines) — raw I/O.
- `acpRpcDispatcher.ts` (220 lines) — pure RPC handler (the renamed `GeminiAgent`).
- `acpSessionManager.ts` (324 lines) — concurrent user sessions.
- `acpSession.ts` (replacing the old 865-line file with 19 lines) — individual chat turns.
- `acpUtils.ts` (379 lines) — shared helpers and schemas.

Three new test files (`acpRpcDispatcher.test.ts` 338 lines, `acpSessionManager.test.ts` 386 lines, `acpSession.test.ts` 463 lines) take the place of the giant test. Five existing files in the dir get the `acp` prefix (e.g. `commandHandler.ts` → `acpCommandHandler.ts`); these are 0/0 line diffs so it's just renames. `gemini.tsx` import path updated.

## Design analysis

The five-way split matches the natural seams: transport vs dispatch vs session-manager vs per-session vs utils. That's the standard layering for any RPC client and the right one. The line counts also line up: dispatcher is the busiest (220 LoC), per-session is now a thin coordinator (19 LoC) which suggests the heavy session logic moved into the manager and the per-turn lifecycle was where the old monolith was hiding most of its complexity.

A few things worth scrutinising in a refactor of this size:

1. **Pure rename vs behaviour change.** PRs of this shape almost always smuggle one or two opportunistic behaviour changes alongside the moves. Run `git diff main...HEAD -- packages/cli/src/acp/acpRpcDispatcher.ts | grep -E '^[+-]' | grep -vE '^\+\+\+|^---'` and check the *non-rename* lines — anything that isn't a moved function body is suspicious. The PR claims it's a refactor; the test count change should be neutral, not +N/-M.

2. **Test coverage parity.** The old test file was 2294 lines; the three new ones sum to 1187 lines. That's roughly half. Either the old file had massive duplication that the split exposed (plausible — long monoliths grow copy-paste), or some test scenarios got dropped. Worth asking the author to map: which tests in the old file map to which test in the new files, and which (if any) were intentionally deleted as redundant. A coverage-percentage delta from CI would also be reassuring.

3. **File naming consistency.** The `acp` prefix on every file in the `acp/` directory is redundant — the directory already qualifies the namespace. `acp/commandHandler.ts` reads cleaner than `acp/acpCommandHandler.ts`. The PR justifies it as "avoid name collisions" but the only collision risk is at the import-symbol level, not the filename level (TypeScript imports resolve by path). I'd push back on the prefix in code review, but this is a taste call and the maintainers may have their reasons.

## Risks

For a refactor of 16 files and ~5400 line-deltas, the risk surface is non-trivial. The main concerns:

- **Dynamic dispatch / runtime imports.** If anything in the codebase did `await import('./acpClient')` or referenced the file by string, the rename breaks at runtime, not at compile. Worth grepping the whole repo for `acpClient` and `commandHandler`/`fileSystemService` string literals.
- **External consumers.** Anything that imports from these paths in `@google/gemini-cli` consumer packages (extensions, Zed/JetBrains integrations referenced in the manual test list) breaks. The PR mentions Zed/JetBrains testing but only checks the `npm run` flow on macOS — Windows and Linux boxes are unchecked.
- **Test split losing scenarios.** See point 2 above. This is the highest-value review check.

## Verdict

**Verdict:** request-changes

Two blocking asks: (a) provide a mapping of old `acpClient.test.ts` test cases to the new test files showing nothing was dropped silently, with a coverage-delta number from CI; (b) cross-platform validation (at least Linux) before merging a refactor that affects the ACP transport layer used by external editor integrations. The architectural direction is sound and I'd happily approve once those two are addressed.

---

*Reviewed by drip-155.*
