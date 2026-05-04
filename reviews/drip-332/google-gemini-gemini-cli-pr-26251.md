# google-gemini/gemini-cli PR #26251 — Update review-frontend.toml

- **Link:** https://github.com/google-gemini/gemini-cli/pull/26251
- **Head SHA:** `d54e51a617ae2ed8896c30cdac124ef303c909e3`
- **Author:** warrior205 (Naka Moe Toe)
- **Created:** 2026-04-30
- **Files:** `.gemini/commands/review-frontend.toml` (+5 / −0)
- **Diff size:** 13 lines

## Summary
This PR prepends five lines to the project's TOML command file. The added lines are **not valid TOML** and look strongly like a malformed attempt at injection or vandalism, not a legitimate config change.

## Specific citations
- `.gemini/commands/review-frontend.toml:1-5` (added):
  ```
  import clang
  import dotnet
  import pwsh
  \\\\\\\\\\
  ```
  followed by the existing TOML body (`description=...`, `prompt = """..."""`).

## Why this should not merge
1. **Syntactically invalid.** TOML has no `import` statement. The first three lines will cause the TOML parser to fail at load time, breaking the `review-frontend` command entirely. There is no plausible upstream-tooling interpretation of `import clang` in a TOML config.
2. **The `\\\\\\\\\\` line is a bare escape sequence with no key/value context.** TOML parsers will reject this immediately.
3. **No tests, no description rationale, no maintainer dialogue justifies the change.**
4. **Pattern matches a known low-effort injection vector** — pasting plausible-looking tokens (`clang`, `dotnet`, `pwsh`) into a prompt-related file in the hope that the agent reading it later will treat them as instructions. The fact that this lands in a *prompt-defining* TOML (`prompt = """..."""` directly below) makes the intent more concerning, not less. Even if benign, this is the exact shape that a prompt-injection PR takes.

## Recommended actions
- Close as request-changes with a note pointing the author at the TOML spec.
- If the author intended to extend the prompt, they should edit inside the `prompt = """..."""` block, not above it.
- Maintainer should consider a CI step that validates `.gemini/commands/*.toml` parses cleanly before allowing merge — this PR would be auto-blocked by even a `tomllib.loads()` smoke test.

## Verdict
`request-changes`
