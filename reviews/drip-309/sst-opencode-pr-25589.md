# sst/opencode #25589 — docs: fix markdown table rendering in experimental features (zh-CN)

- PR: https://github.com/sst/opencode/pull/25589
- Author: dk363
- Head SHA: `3b298f28ec0bc4c170a035525a7c7d307793e94e`
- Updated: 2026-05-03T13:13:34Z

## Summary
Fixes a broken Markdown table in the Chinese docs (`packages/web/src/content/docs/zh-cn/cli.mdx`) for the experimental environment-variable section. The header row had been written with extra `| --- | ... |` spans (4 columns of separators) while body rows were 3 columns, breaking the table renderer. One row was also collapsed onto a single line with two entries via stray `|` separators.

## Observations
- `packages/web/src/content/docs/zh-cn/cli.mdx:599`: the broken header `| ----- | ----- | ----- | --- | ----- | ----- | ----- |` is replaced with the correct 3-column `| ----- | ----- | ----- |`. Correct fix.
- `packages/web/src/content/docs/zh-cn/cli.mdx:610-612`: the squashed line `| OPENCODE_EXPERIMENTAL_LSP_TY ... | ... | OPENCODE_EXPERIMENTAL_MARKDOWN | boolean | ... |` is split into two clean rows. Correct.
- This change matches the structure of the English source (`packages/web/src/content/docs/cli.mdx`) for the same section — consider verifying parity by diffing the en and zh-cn header rows; if the English version was the source of truth and got out of sync with zh-cn, the same row order and the same set of variables should now match. From the diff alone, both `OPENCODE_EXPERIMENTAL_MARKDOWN` and `OPENCODE_EXPERIMENTAL_LSP_TY` are present in zh-cn now; spot-check that no entry is *missing* relative to en.
- Pure docs/zh-CN change, no risk to runtime. No tests needed.
- Nit: the PR title has a trailing space ("(zh-CN) "). Harmless but trivially fixable.

## Verdict
`merge-as-is`
