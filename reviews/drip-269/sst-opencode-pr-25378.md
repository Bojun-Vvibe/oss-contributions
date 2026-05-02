# sst/opencode #25378 — docs(ecosystem): add constellation-opencode

- URL: https://github.com/sst/opencode/pull/25378
- Head SHA: `25d34b588d719ff8ac10ead72a7d5f962333eec2`
- Author: @rbonestell
- Stats: +1 / -0 across 1 file

## Summary

Single-line docs change adding the constellation-opencode plugin to the
ecosystem table.

## Specific feedback

- `packages/web/src/content/docs/ecosystem.mdx:55` — new row inserted in
  alphabetical-ish order following the existing pattern. Pipe alignment
  matches the surrounding rows. Description "Upgrade OpenCode from text
  search to code understanding" is concise and matches the project's
  marketing copy at https://docs.constellationdev.io/plugins/opencode.
- The link target `https://github.com/ShiftinBits/constellation-opencode`
  resolves to a real repo; no link-rot risk at submission time.
- Strictly speaking the table sort order in this file is informal (entries
  are loosely chronological), so insertion position is fine.
- No code, no tests required. README/CHANGELOG not gated for ecosystem
  table additions in this repo's history.

## Verdict

`merge-as-is` — trivial, correct, low risk.
