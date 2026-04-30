# google-gemini/gemini-cli#26251 — Update review-frontend.toml

- URL: https://github.com/google-gemini/gemini-cli/pull/26251
- Head SHA: `d54e51a617ae2ed8896c30cdac124ef303c909e3`
- Size: +5 / -0 (1 file)

## Summary

Single-file diff against `.gemini/commands/review-frontend.toml` that prepends five lines: `import clang`, `import dotnet`, `import pwsh`, a line of literal backslashes (`\\\\\\\\\\`), and a blank line, ahead of the existing `description=` and `prompt =` keys. The TOML grammar does not have an `import` statement; `\\\\\\\\\\` is not a comment, key, or table header. The file as written will fail to parse on the first non-comment, non-whitespace line.

## Observations

- `.gemini/commands/review-frontend.toml:1-4` — `import clang`, `import dotnet`, `import pwsh` are not TOML. There is no `import` keyword in the TOML 1.0.0 specification; these lines will produce a parser error at the loader (likely "expected `=` after key" or "invalid bare key"). This change breaks the `/review-frontend` slash command for every user.
- `.gemini/commands/review-frontend.toml:4` — `\\\\\\\\\\` (10 backslashes) is not valid TOML at any nesting. Even if treated as a bare key it would be rejected; if interpreted as the start of a string it lacks delimiters.
- The PR title is "Update review-frontend.toml" with no body description visible in the metadata. There is no test, no rationale, and no linked issue. The change does not pattern-match any plausible refactor, prompt-tuning, or feature for `gemini-cli`'s frontend-review slash command.
- Pattern matches a class of low-quality / accidental / spam PRs that periodically show up against high-traffic repos. Recommend closing with a polite note rather than nit-merging — there is nothing here to salvage.

## Verdict

**request-changes**

## Nits

- The entire diff should be reverted. If the author intended to add file-import scaffolding for the prompt template, that's not how `.gemini/commands/*.toml` works — those are TOML files consumed by gemini-cli's slash-command loader, not source files.
- If the author was testing a prompt-injection or template-include feature, it should be proposed as an issue/RFC against the `commands/*.toml` schema first.
- A CI lint that runs `tomli`/`toml` parse over `.gemini/commands/*.toml` on every PR would catch this class of regression at submit time.
