# charmbracelet/crush #2767 — chore: update `view` tool limit to 200KB

- PR: https://github.com/charmbracelet/crush/pull/2767
- Head SHA: `ca9d7ebea73e6ae2601202ad9d142433ddb5649f`
- State: merged
- Diff size: +4495 / -3205 across 15 files (the bulk is regenerated
  test fixtures/cassettes)

## Summary

Doubles the `view` tool's max file size from 100KB → 200KB. The
production change is a single constant + a one-word docstring update;
all other diff lines are regenerated VCR-style YAML fixtures under
`internal/agent/testdata/TestCoderAgent/glm-5.1/*.yaml` because the
docstring is embedded into recorded LLM prompts.

## Citations

- `internal/agent/tools/view.go:56` —
  `MaxViewSize = 200 * 1024 // 200KB` (was `100 * 1024`). One line,
  no other call-site adjustments needed because `MaxLineLength`
  (2000) and `DefaultReadLimit` (2000) are independent.
- `internal/agent/tools/view.md:1` and `:21` — user-facing doc
  string updated consistently in both the description and the
  `<limitations>` section.
- 13 fixture YAMLs under
  `internal/agent/testdata/TestCoderAgent/glm-5.1/` are wholesale
  regenerated. I spot-checked `bash_tool.yaml` and `download_tool.yaml`
  — the only meaningful change in the embedded system prompts is
  `max 100KB` → `max 200KB`. Recorded HTTP responses (durations,
  chatcmpl IDs) also changed because the cassettes were re-recorded;
  this is expected for VCR-style tests but makes the diff noisy and
  reviewer-hostile.

## Risk / nits

- No memory/perf analysis in the PR. 200KB per tool call × N
  concurrent agents × multi-message context = noticeable token
  pressure. At typical model token rates (~4 chars/token), 200KB is
  ~50K tokens — a single `view` can now consume a large chunk of a
  128K-context model's window. Worth a comment in the docstring
  pointing users at `offset`/`limit` for big files.
- Re-recorded cassettes from a single model (glm-5.1) — fixture
  drift across providers/models was not addressed. Probably out of
  scope for this PR but worth noting.
- The docstring change inside fixture YAMLs is duplicated dozens of
  times. Future docstring edits will keep producing massive diffs;
  consider a fixture-templating approach, but again, out of scope.

## Verdict

`merge-as-is` — already merged, and this is a one-constant change
with mechanically-regenerated fixtures. The token-budget concern is
worth a follow-up doc note, not a block.
