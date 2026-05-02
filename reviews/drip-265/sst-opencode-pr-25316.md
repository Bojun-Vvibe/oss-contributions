# sst/opencode #25316 — fix: sanitize malformed agent frontmatter

- **Repo:** sst/opencode
- **PR:** #25316
- **Head SHA:** `353c7b519e253c6f9f8c8042d5550af0e533dc08`
- **Verdict:** merge-after-nits

## Summary

Fixes a real visibility bug: when a subagent's `description` contains an
unquoted colon (e.g. `Reviews changes: returns a verdict.`), `gray-matter`
silently returns empty `data` and treats the entire file as content. The
hidden agent then leaks into the Web UI as a fake top-level prompt, with
`mode`/`hidden` lost.

Three changes in `packages/opencode/src/config/markdown.ts`:

1. The frontmatter regex (markdown.ts:20) is rewritten to capture
   leading/trailing newlines (`(\r?\n)`) and require the closing `---`
   to be at the start of a line (`^---[ \t]*(\r?\n|$)/m`). Necessary
   because the old regex's `replace(frontmatter, ...)` could collide
   with body text containing the same substring.
2. `processed = result.join(match[1])` (markdown.ts:65) preserves the
   original line ending style (CRLF vs LF).
3. `parse()` (markdown.ts:74-76) now retries with `fallbackSanitization`
   when `matter()` returns empty `data` but the sanitized form differs —
   covering the silent-failure case `try/catch` doesn't catch.

## Specific notes

- **Nit (markdown.ts:20):** the regex uses `^---` anchored with `/m`
  but the leading `^---` (start of file) is implicit on the first
  match. Worth adding a comment that this regex assumes frontmatter is
  the first thing in the file — `parse()` calls it on raw template
  content, which is fine, but a future caller passing arbitrary
  markdown could match a later `---` block.
- **Nit (markdown.ts:75):** the silent-retry runs `fallbackSanitization`
  unconditionally on every parse, even when `md.data` is non-empty.
  That's wasteful but cheap; consider moving the call inside the
  `if (Object.keys(md.data).length === 0)` branch — same semantics,
  no work on the hot path.
- **Test coverage:** the new fixture `colon-in-value.md` plus
  `markdown.test.ts:233-296` cover all the metadata fields and the
  end-to-end `ConfigAgent.load()` path. Solid.

## Rationale

Real bug with a clean reproducer, narrow fix, good regression test.
The two nits are quality-of-life and shouldn't block merge.
