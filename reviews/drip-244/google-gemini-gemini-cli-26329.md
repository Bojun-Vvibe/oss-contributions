# PR #26329 — fix(cli): undeprecate --prompt and correct positional query docs

- Repo: google-gemini/gemini-cli
- Head: `d2aca8d8230e95846238af8e88e0f72b05695650`
- URL: https://github.com/google-gemini/gemini-cli/pull/26329
- Verdict: **request-changes**

## What lands

Three doc-only edits intended to fix #19156:

1. `docs/reference/configuration.md:2581` — removes the line `**Deprecated:**
   Use positional arguments instead.` from the `--prompt` / `-p` entry.
2. `docs/cli/cli-reference.md:12` — changes the description for the
   `gemini "query"` example.
3. `packages/cli/src/config/config.ts` — referenced in PR body as receiving
   help-text update (not in the visible diff snippet but called out by author).

## The blocker

`docs/cli/cli-reference.md:12` in this diff replaces:

```
| `gemini "query"`                   | Query and continue interactively   | `gemini "explain this project"`                              |
```

with:

```
| gemini "query"                   | Query and continue interactively   | gemini "explain this project"                              |
```

**The backticks were lost.** This is a markdown table cell where surrounding
cells use `\`gemini …\`` to render as inline code. The change strips backticks
from both the command column and the example column, breaking the visual
consistency of the table and causing the cells to render as plain text instead
of monospaced code. Inspect the raw diff:

```diff
-| `gemini "query"`                   | Query and continue interactively   | `gemini "explain this project"`                              |
+| gemini "query"                   | Query and continue interactively   | gemini "explain this project"                              |
```

Worse, the description `Query and continue interactively` is **not changed**
in this diff hunk — the author's stated intent was to fix the description to
say "non-interactive" (per the PR body: *"Updated docs/cli/cli-reference.md to
show gemini "query" as non-interactive"*). The actual diff only mangles
formatting and leaves the misleading description in place. So the PR achieves
**neither** of its stated goals on this file:

- The description is unchanged.
- The formatting is regressed.

## Other observations

- The `docs/reference/configuration.md` deletion of the deprecation line is
  fine in isolation, but if `--prompt` was deprecated by an earlier PR with
  intent (rather than incorrectly), undeprecating it via doc-only change
  silently disagrees with whatever the earlier author intended. The PR body
  doesn't reference the deprecation's introducing PR or rationale — a brief
  "deprecated in #X for reason Y, no longer relevant because Z" would close
  the loop.
- The `packages/cli/src/config/config.ts` change isn't in the visible diff but
  is mentioned in PR body. If it's just help-text, fine, but it's worth
  including in the same review pass.
- The actual fix for #19156 (positional args starting with `-` failing because
  yargs parses them as options, e.g. `gemini "-1+2=?"`) is **not** addressed
  by this PR. The PR body acknowledges this but the fix is just "use
  `--prompt` instead, and we'll undeprecate it." That's a workaround
  documented as a fix; the underlying yargs behavior could be fixed with
  `parserConfiguration: { 'unknown-options-as-args': true }` or similar. Not
  a blocker for *this* PR but worth a follow-up issue.

## Required changes before merge

1. Restore the backticks around `gemini "query"` and `gemini "explain this
   project"` in `docs/cli/cli-reference.md:12`.
2. Actually change the description for the `gemini "query"` row from
   `Query and continue interactively` to the non-interactive language the PR
   body claims.
3. Confirm `packages/cli/src/config/config.ts` help-text change is included
   and consistent with the doc updates.

After those three are addressed this becomes a clean merge-as-is — the intent
is sound, the execution on this file is broken.