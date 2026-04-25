# cline/cline PR #10242 — fix: quote file mention paths containing spaces

@f5945553 · base `main` · +34/-3 · author `bob10042`

## Summary
Fixes the "Add to Cline" generation side of file mentions so paths containing spaces get wrapped in the existing quoted-mention syntax (`@"/path with spaces"`). Parsing already supported quoted mentions (upstream commit `0d933e8`); this is the missing emit-side fix. Closes #7789.

## What changed
- `src/core/mentions/index.ts:50-56` — `getFileMentionFromPath` now delegates to a new `formatFileMention` helper for both the no-cwd and relative-path branches.
- `src/core/mentions/index.ts:58-65` — new `formatFileMention(filePath)` normalizes backslashes to forward slashes, ensures a leading `/`, then returns `@"/x"` if the path contains a space, otherwise `@/x`.
- `src/core/mentions/index.test.ts:14-40` — three new test cases: simple relative mention (no space), relative mention with spaces (asserts `@"/path with spaces/file.txt"`), and absolute mention with no cwd available.

## Key observations
- The `mentionPath.includes(" ")` check at line 64 only covers ASCII space — tab characters or other whitespace that the LLM/Tokenizer might split on are not quoted. Probably fine in practice (tabs in paths are vanishingly rare on macOS/Linux/Windows file managers) but worth a comment.
- The backslash normalization at line 62 (`replace(/\\/g, "/")`) is correct for emitting Unix-style mentions on Windows; matches what the parser side expects.
- Tests stub `HostProvider.workspace.getWorkspacePaths` correctly; assertions are on the exact emitted string which is what downstream parsing depends on.
- Absolute-path branch (`if (!cwd)`) now also normalizes/quotes — a nice generalization beyond the issue.
- Tiny: `mentionPath.startsWith("/")` check is redundant for the relative branch since `path.relative(cwd, filePath)` never returns a leading `/`; harmless but the conditional is for the no-cwd branch.

## Risks/nits
- No risk to parsing because the parser already handles both forms; this strictly improves output fidelity.

**Verdict: merge-as-is**
