# sst/opencode PR #24008 — fix: preserve newlines in ACP command argument parsing

- **Author:** feisuzhu
- **Head SHA:** 29d54e5e2db970bfcc1ece6dd1ff182f1758f5cd
- **Files:** `packages/opencode/src/acp/agent.ts` (+3 / −2)
- **Verdict:** `merge-as-is`

## Context

ACP (Agent Client Protocol) lets the user issue slash commands like
`/edit foo\n  with multiline body`. The previous parser tokenised on
`/\s+/` and re-joined with a single space, which silently collapsed
newlines, runs of spaces, and tabs in the argument body. For commands
that hand the args off as a free-form prompt, that destroys
formatting (Markdown blocks, code fences, indented bullets).

## What the diff does

At `packages/opencode/src/acp/agent.ts:1448-1453` the parser is
rewritten:

```ts
- const [name, ...rest] = text.slice(1).split(/\s+/)
- return { name, args: rest.join(" ").trim() }
+ const match = text.slice(1).match(/^(\S+)(?:\s([\s\S]*))?$/)
+ if (!match) return
+ return { name: match[1], args: (match[2] ?? "").trim() }
```

The regex now anchors the command name as the leading run of
non-whitespace, takes a single whitespace separator, and captures the
remainder verbatim with `[\s\S]*` (DOTALL-equivalent for the args
body). `.trim()` strips leading/trailing whitespace but keeps internal
newlines intact.

## Review notes

- Correct shape: `^(\S+)(?:\s([\s\S]*))?$` cleanly separates "name
  required, args optional, args may be multiline." The optional
  non-capturing group handles `/help` (no args) without producing an
  empty `args:""` accidentally — `match[2]` is `undefined` and the
  `?? ""` fallback handles it.
- The single-`\s` separator is right. Greedy `\s+` would re-introduce
  the bug for users who wrote `/edit␣␣␣foo` — they'd lose the
  significant leading whitespace on the first line of args. With this
  regex, only one separator char is consumed.
- The `if (!match) return` guard means `text.slice(1) === ""` (i.e.
  user typed just `/`) now returns `undefined` instead of
  `{ name: "", args: "" }`. That's an improvement: callers that key
  on `name` no longer dispatch on the empty string.
- No new tests in the diff. Given how subtle whitespace handling is,
  three table-driven cases (`/cmd`, `/cmd arg`, `/cmd \n  multi\n  line`)
  would lock in the contract cheaply. Not a blocker.

## What I learned

The classic `split / re-join` idiom is information-lossy whenever the
delimiter run carries semantic content. Anchored regex with explicit
optional tail (`(?:\s(.+))?`) is the safer pattern when only the
*first* whitespace boundary matters and everything past it is opaque
to the parser.
