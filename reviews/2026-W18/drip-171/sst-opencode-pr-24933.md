# sst/opencode#24933 — `fix(cli): detect MIME type for --file flag instead of hardcoding text/plain`

- URL: https://github.com/sst/opencode/pull/24933
- Head SHA: `0d1dade4d0d7`
- Author: rogerdigital
- Size: +1 / -1 in `packages/opencode/src/cli/cmd/run.ts`
- Closes: #24698

## Summary

`opencode run --file <path>` always tagged the attached file with
`text/plain`, regardless of actual content. That caused image
attachments (and any other binary) to be passed to the model as if
they were UTF-8 text, defeating multimodal routing on providers that
key off the MIME type. The fix replaces the hardcoded literal with
the existing `Filesystem.mimeType()` helper.

## Specific reference

`packages/opencode/src/cli/cmd/run.ts:331` (line 328 in the diff
context, post-patch line 331):

```ts
- const mime = (await Filesystem.isDir(resolvedPath)) ? "application/x-directory" : "text/plain"
+ const mime = (await Filesystem.isDir(resolvedPath)) ? "application/x-directory" : await Filesystem.mimeType(resolvedPath)
```

The directory branch is preserved; only the non-directory fallback
changes. `Filesystem.mimeType()` already exists and is the same
helper used by other ingestion paths, so this is the obvious
single-line fix.

## Risks

- `mime-types` returns `false` for unknown extensions. The follow-up
  question is what `Filesystem.mimeType()` does in that case — if it
  falls back to `application/octet-stream`, multimodal-incapable
  providers may now reject what used to silently work as text.
  Worth a one-liner test for "unknown extension → reasonable
  fallback (text/plain or octet-stream consistent with other call
  sites)".
- No new test added. For a fix that's literally about wrong MIME
  classification, a small unit test exercising at least
  `image/png`, `text/plain` (e.g. `.txt`), and unknown extension
  would lock the behavior.

## Verdict

`merge-after-nits` — add a regression test or at minimum a snippet
in the PR description verifying the three cases above. The change
itself is correct and minimal.

## Suggested nits

- Add a `run.test.ts` case for `--file foo.png` asserting
  `mime === "image/png"`.
- Confirm `Filesystem.mimeType()`'s unknown-extension behavior in
  the PR description so reviewers don't have to chase it.
