# block/goose PR #8974 — Add VC Deal Flow Signal MCP to extensions directory

- URL: https://github.com/block/goose/pull/8974
- Head SHA: `ee5e4e1d08f7db4cdd7b87fe29b7aee82ea2382c`
- Verdict: **merge-after-nits**

## Summary

Pure data-file change. Appends one new entry to
`documentation/static/servers.json` registering an external MCP server
("VC Deal Flow Signal", `npx -y @gitdealflow/mcp-signal`,
`is_builtin: false`, `endorsed: false`). The same diff also rewrites
the unicode characters `⚡` and `—` into their `\u26a1` / `\u2014`
escape forms in two unrelated entries (Apify and Cash App), which
suggests the file was re-serialized by a JSON formatter that escapes
non-ASCII.

## Specific references

- `documentation/static/servers.json` lines 36-39 — the Apify entry's
  `description` field changes only from a literal `⚡` to `\u26a1`.
  Identical glyph; cosmetic.
- `documentation/static/servers.json` lines 848-851 — the Cash App
  entry's `description` em-dash (`—`) becomes `\u2014`. Identical
  glyph; cosmetic.
- `documentation/static/servers.json` lines 856-867 — the new
  `vc-deal-flow-signal` entry. Includes `id`, `name`, `description`,
  `command`, `link` (GitHub), `installation_notes`, `is_builtin: false`,
  `endorsed: false`, `environmentVariables: []`.

## Commentary

For an extensions-registry PR there are really only three things to
check, and this one passes two of them:

1. **Schema conformance.** The new entry has the same key set as the
   surrounding entries (`environmentVariables: []` is correct for an
   auth-less server, `endorsed: false` is the right default for an
   unfamiliar third-party submission). ✓

2. **Linkability.** The `link` points at
   `https://github.com/kindrat86/mcp-deal-flow-signal`, which a
   reviewer should manually open and confirm: (a) repo exists, (b)
   the `npx -y @gitdealflow/mcp-signal` command actually invokes
   that codebase, (c) the package is published under that name on
   npm. The PR description should ideally include those confirmation
   steps; the diff alone can't prove them.

3. **Trust posture.** `endorsed: false` correctly signals "we haven't
   vetted this." Good. The description claims "No auth required" and
   "read-only tools" — fine, but a one-line review note from the
   author confirming they've actually run the server and seen those
   tool calls fire would be appreciated.

Nits before merge:

- **The stray unicode-escape rewrites.** The Apify and Cash App
  edits change the on-disk byte representation but not the rendered
  glyph. They're harmless, but they pollute the diff and make it
  harder to spot if someone smuggled a real semantic change into
  another existing entry. Worth either (a) reverting to leave those
  entries byte-identical or (b) splitting the formatter pass into a
  separate commit.

- **Missing newline check.** Confirm the file still ends with a
  trailing newline; the diff hunk shows the new entry as the last
  array element, so an automated formatter run would have caught
  this but a manual edit might not have.

Substance is fine. Cosmetic split would make the diff cleaner. Merge
after the formatter rewrites are split out (or accept them and merge
as-is if the maintainers don't mind the noise).
