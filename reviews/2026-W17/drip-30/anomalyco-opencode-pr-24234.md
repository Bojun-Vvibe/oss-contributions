# anomalyco/opencode#24234 — fix(config): detect non-object frontmatter data from gray-matter

**Verdict: merge-as-is**

- PR: https://github.com/anomalyco/opencode/pull/24234
- Author: kagura-agent
- Base SHA: `386091b79a76ebc49ce2eba9628a2f93ce0061ee`
- Head SHA: `bf93af352960a192303a9fd2398771a398b2610f`
- +52 / -1

Closes #24181.

## Summary

`gray-matter` silently returns `md.data` as a non-object (string,
number, boolean, array) when the YAML frontmatter is malformed — for
example `skill:true` (no space after colon) parses as the string
`"skill:true"`. The downstream `...md.data` spread in `agent.ts` then
produces garbage indexed properties (`{0:'s', 1:'k', ...}`). If
`safeParse` still passes (because `name` from filename and `prompt`
from content are valid), the agent loads with degraded config — no
tools, no permissions, no model override — with zero error feedback.
The fix adds a type guard in `ConfigMarkdown.parse()` after both the
initial `matter()` call and the fallback sanitization pass; on
non-object data it raises and the catch arm rethrows as
`FrontmatterError` with the offending file path.

## Specific findings

- `packages/opencode/src/config/markdown.ts:72` (head SHA
  `bf93af352960a192303a9fd2398771a398b2610f`):
  ```ts
  const md = matter(template)
  // gray-matter may return md.data as a string instead of an object when YAML
  // is malformed (e.g. "skill:true" without space after colon). Treat this as
  // a parse failure so the fallback sanitizer gets a chance to fix it.
  if (typeof md.data !== "object" || md.data === null || Array.isArray(md.data)) {
    throw new Error("frontmatter parsed as non-object")
  }
  return md
  ```
  The triple guard (`typeof !== "object"`, `=== null`, `Array.isArray`)
  is the correct shape because in JS `typeof null === "object"` and
  `typeof [] === "object"`. Missing any of those would let `null` or
  arrays through and re-trigger the original spread bug. The comment
  documenting the gray-matter behavior is useful future-proofing.
- Same file, the fallback path at `:79`:
  ```ts
  const fallback = matter(fallbackSanitization(template))
  if (typeof fallback.data !== "object" || fallback.data === null || Array.isArray(fallback.data)) {
    throw new Error("frontmatter parsed as non-object after sanitization")
  }
  return fallback
  ```
  Distinct error message ("after sanitization") aids debugging. The
  outer `catch (err)` then throws `FrontmatterError` with the
  triggering file path, so the user gets a pointer not just a generic
  "frontmatter broken".
- `packages/opencode/test/config/markdown.test.ts:211` — two new
  tests. The first uses a committed fixture
  `fixtures/invalid-yaml-frontmatter.md` containing `skill:true`. The
  second is a parameterized loop covering all four non-object types
  (number `42`, boolean `true`, array `- item`, string `just text`)
  — each writes a temp file, calls `parse`, and asserts
  `FrontmatterError.isInstance(err)`. The temp-file cleanup in
  `finally { try { unlinkSync(tmp) } catch {} }` is correct.
- `packages/opencode/test/config/fixtures/invalid-yaml-frontmatter.md` —
  new fixture, 5 lines, exactly the `skill:true` repro case from
  issue #24181.

## Rationale

Tight, well-scoped fix for a silent-failure class bug — the worst
kind, where the user thinks their agent config is loaded but it's
actually running with empty tools/permissions/model and there's no
error to point at. The type-guard approach is correct: just trying
to fix `md.data` after-the-fact via the spread is what produced the
garbage in the first place. By treating non-object as a parse
failure, the fallback sanitizer gets a chance to repair common YAML
mistakes (the `fallbackSanitization` helper is presumably the
existing "add space after colon" / similar fixer), and only when
that *also* fails does the user see `FrontmatterError`. Both layers
of the parse pipeline get the guard, which is important — without
the post-fallback guard, a sanitization path that produces a non-object
result would silently regress to the same broken state. The
parameterized test covering number/boolean/array/string is exactly
the right shape: gray-matter has historically returned each of these
for different malformed YAML patterns, so a regression that fixed
only one type would still leak. The 39-test count in the PR body
suggests the existing markdown.test.ts suite is healthy and the new
cases slot in cleanly. No nits — comment quality is good, error
messages are distinct, fixture is minimal. Merge.
