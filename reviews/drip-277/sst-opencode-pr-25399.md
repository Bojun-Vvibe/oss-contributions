# Review — sst/opencode#25399

- PR: https://github.com/sst/opencode/pull/25399
- Title: fix(docs): CLI docs for current commands and flags
- Head SHA: `17956e667287aeaa0e217a4570c66063a41c542a`
- Size: +1676 / −1413 across 18 files
- Verdict: **merge-after-nits**

## Summary

Pure docs PR that resyncs the localized `cli.mdx` pages (ar, de, es,
fr, ja, ko, pt-br, ru, tr, zh-cn, plus the en source) with the actual
CLI surface. Two real changes are folded in: (1) every flag cell in
every locale is wrapped in `<nobr><code>{"--flag"}</code></nobr>` so
that long flag names like `--continue` / `--password` don't soft-wrap
in the rendered table column, and (2) several previously-undocumented
flags (`--continue`, `--fork`, `--password`, `--username`, `--dir`,
`--variant`, `--thinking`) are added to the `attach` and `run`
command tables.

## Evidence

- `packages/web/src/content/docs/ar/cli.mdx` (and the 10 sibling
  locale files) — the `attach` flag table grows from 2 rows
  (`--dir`, `--session`) to 6 rows, picking up `--continue`,
  `--fork`, `--password`, `--username`. The `run` table picks up
  `--password`, `--username`, `--dir`, `--variant`, `--thinking`.
- The escape `<code>{"--continue"}</code>` (JSX expression with a
  literal string) is the standard MDX pattern to keep `--` from being
  interpreted as a typographic en-dash by the markdown renderer. It
  was previously inconsistent across locales; this PR makes it
  uniform.
- The diff is huge by line count but trivially auditable: the
  underlying English copy is unchanged in semantic content, and the
  per-locale wrapping is a mechanical transform.

## Notes / nits

- The `<nobr>` element is non-standard HTML and not in the HTML
  Living Standard. It does work in every current browser engine, and
  Astro/MDX will pass it through, so it's fine in practice — but a
  follow-up that swaps to `style="white-space: nowrap"` on the
  surrounding `<code>` (or a CSS class on the table cell) would be
  more future-proof. Not a blocker.
- The Arabic, Hebrew-style RTL locale (`ar`) keeps the table column
  alignment markers (`---`) intact; double-check that the rendered
  RTL table still aligns the flag column on the right. Visually
  verifying one rendered locale before merge would catch any pipe-
  alignment regression.
- Two flag descriptions were translated inconsistently (e.g. the
  Arabic `--variant` row reads "متغير النموذج (جهد الاستدلال الخاص
  بالمزود)" — fine — but the matching ru/tr rows could be spot-
  checked by a native reviewer if one is around).

Ship after a quick visual on one rendered RTL locale.
