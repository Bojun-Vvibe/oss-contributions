# QwenLM/qwen-code#3835 — feat(sdk-python): replace verbatim release notes inheritance with --generate-notes

- PR ref: `QwenLM/qwen-code#3835` (https://github.com/QwenLM/qwen-code/pull/3835)
- Head SHA: `0d72c8d4476101e7ebc1d75848e24e730ad47853`
- Title: feat(sdk-python): replace verbatim release notes inheritance with --generate-notes
- Verdict: **merge-after-nits**

## Review

The diff in `.github/workflows/release-sdk-python.yml:403-424` removes the brittle
"fetch previous release body via `gh release view ... --json body -q .body`, then
re-paste it verbatim" pattern and replaces it with `gh release create
--generate-notes --notes-start-tag sdk-python-${PREVIOUS_RELEASE_TAG}`. This is
strictly better: previous-notes-fetching had a hand-rolled error case-table for `"release
not found" / "Not Found" / "HTTP 404"` that would silently fall back to "See commit
history for changes." on any mismatch in `gh`'s error string formatting, which made the
pipeline's failure mode invisible.

The `--notes-start-tag` flag is exactly the right tool here: it tells GitHub's
auto-notes generator to scope the diff to commits since the prior SDK tag (rather than
since the most recent tag in the repo, which would mix in non-SDK commits). The
fallback at line 415-419 — when there's no prior SDK tag, blank `GENERATE_NOTES_FLAG`
and write the static "Initial release. See commit history for changes." line — is the
correct behavior for an initial release, since auto-notes against no start tag would
indeed include unrelated history.

Nits:
1. The unquoted variable expansions at `release-sdk-python.yml:421-422`
   (`${GENERATE_NOTES_FLAG}` and `${NOTES_START_TAG_FLAG}`) rely on shell word-splitting
   to break `--notes-start-tag sdk-python-X.Y.Z` into two args. That works but trips
   shellcheck SC2086 and is fragile if the tag ever contains whitespace (it can't here,
   but the pattern is foot-gun-shaped). Consider switching to a bash array:
   `extra_flags=(--generate-notes --notes-start-tag "sdk-python-${PREVIOUS_RELEASE_TAG}")`
   then `gh release create ... "${extra_flags[@]}"`.
2. The header text appended to `${NOTES_FILE}` earlier in the script is now followed by
   GitHub's auto-generated section, which means the final release body has a
   hand-written preamble + machine-generated changelog. Worth verifying the visual
   ordering on a dry-run release before this lands on a public tag — the current diff
   doesn't show a separator/heading between the two sections.
