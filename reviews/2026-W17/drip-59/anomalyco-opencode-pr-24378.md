# anomalyco/opencode #24378 — docs: sync env vars with source code

- URL: https://github.com/anomalyco/opencode/pull/24378
- Head SHA: `b2fa9170ca449005c5e2991984f5b9010381d8fd`
- State: OPEN
- Size: +1197/-831 across ~20+ translated MDX files
- Closes: #24235
- Verdict: **merge-after-nits**

## What it does

Walks every locale's `cli.mdx` (and `providers.mdx` where relevant)
under `packages/web/src/content/docs/<locale>/` and rewrites the
"Environment Variables" tables to:

1. Match the actual env vars declared in source.
2. Sort them alphabetically (the previous order was insertion-order
   and had drifted — e.g. `OPENCODE_GIT_BASH_PATH` was sandwiched
   between auth/share rows in `ar/cli.mdx:557-580`).
3. Add new entries (`OPENCODE_AUTH_CONTENT`,
   `OPENCODE_DISABLE_EXTERNAL_SKILLS`, `OPENCODE_DISABLE_MOUSE`,
   `OPENCODE_DISABLE_PROJECT_CONFIG`, etc.) that the source code
   exposes but the docs hadn't picked up.

## Diff notes

Reviewing the Arabic file as the reference (`ar/cli.mdx:555-595`):
- New rows added in alphabetical position (e.g.
  `OPENCODE_AUTH_CONTENT` lands above `OPENCODE_AUTO_SHARE`,
  `OPENCODE_CLIENT` before `OPENCODE_CONFIG`).
- Existing rows are physically *moved*, not edited, so the diff is
  mostly delete-then-add of identical-content lines — that's why
  the +1197/-831 spread is so large for what's effectively a sort.
- The English table at `cli.mdx` (the canonical source) is updated
  in lock-step.

The same shape appears across all locale files — this looks like the
output of a doc-generator run rather than hand-editing each
translation.

## Risk surface / nits

- **Translated descriptions for the *new* rows.** Spot-checking
  `ar/cli.mdx`, `OPENCODE_DISABLE_EXTERNAL_SKILLS` is described as
  "تعطيل تحميل المهارات من حزم npm" — that's a passable Arabic
  rendering, but for the new keys (`OPENCODE_AUTH_CONTENT`,
  `OPENCODE_CLIENT`, `OPENCODE_DISABLE_MOUSE`, etc.) the strings
  look machine-generated. Translation reviewers should give them a
  pass — especially `OPENCODE_DISABLE_PROJECT_CONFIG` ("تعطيل
  تحميل التهيئة على مستوى المشروع") which is functionally accurate
  but stylistically uneven against the surrounding rows.
- **No mechanism to keep this in sync going forward.** This PR is
  the "manual snapshot" version of "generate from source". A
  follow-up that wires this generation into a CI doc-check would
  prevent the drift from re-accumulating; otherwise the next env
  var add will silently regress the alignment.
- **Sort key consistency**. Quick spot check shows
  `OPENCODE_DISABLE_*` rows are alphabetised among themselves
  correctly, but the *en* canonical file should be the
  authoritative ordering, and translations should match its row
  order rather than re-alphabetising in the translated language
  (which can produce different orders for non-Latin scripts). The
  PR appears to use the same alphabetical key (`OPENCODE_*`) across
  all locales — good, since the variable name is the join key.
- **No table-of-contents update**. If any translated `cli.mdx` has
  an in-page TOC pointing at section IDs that changed, this PR
  doesn't appear to touch them. Worth scanning.

## Why this verdict

Mechanical-but-correct sync of doc tables to source. The diff is
large but the change is structural (sort + add missing rows). Three
nits: native-speaker pass on the new translated strings, a
suggestion to wire CI-side doc-vs-source check, and a TOC sweep to
make sure no anchors break. None block merge.
