# QwenLM/qwen-code #3654 — refactor: unify tool execution logic across Interactive, Non-Interactive, and ACP modes

- Author: B-A-M-N (John London)
- Head SHA: `84b6f9a01c2206e8c6398efa31741dfec929b909`
- Single file: `TOOL_EXECUTION_UNIFICATION.md` (+1 / −0, new file)

## Specifics

- The PR is currently a **scaffold-only commit**: a new top-level `TOOL_EXECUTION_UNIFICATION.md` containing one literal line (with a `\n\n` shown as escaped characters in the diff, suggesting the file was written via a heredoc that didn't expand newlines) pointing at issue #3247.
- File content (`TOOL_EXECUTION_UNIFICATION.md:1`): `# Tool Execution Unification (Issue #3247)\n\nThis branch tracks the refactoring work to unify tool execution logic across Interactive, Non-Interactive, and ACP modes.\n\nSee Issue #3247 for full context.`
- No code changes, no test changes, no design doc detail beyond a one-line abstract.

## Concerns

- This is a placeholder PR — opening a PR before the refactor lands gives reviewers nothing to evaluate and clutters the open-PR queue. If the author wants early visibility, a draft PR with at minimum a structured design section (current call sites in each mode, target unified API surface, migration plan, test strategy) would be reviewable. As-is, there's no signal.
- The escape-sequence rendering of `\n\n` inside the file body suggests the file was created with `echo -e` or a tool that didn't strip backslash escapes. The committed bytes are likely literal `\n\n` characters, not actual newlines. Author should re-create the file with real newlines or, better, structure it as a markdown design doc.
- Top-level placement (`/TOOL_EXECUTION_UNIFICATION.md`) is unusual for an in-progress design doc — `docs/design/tool-execution-unification.md` would be the conventional spot. The repo also has the in-flight #3583 `docs/design/custom-api-key-auth-wizard-prd.md` PRD precedent that this could mirror.

## Verdict

`request-changes` — convert to draft, fix the literal-`\n` issue, move under `docs/design/`, and either flesh out the design doc to something reviewable or wait until the actual refactor commits exist. There is nothing here to review at the code level.

