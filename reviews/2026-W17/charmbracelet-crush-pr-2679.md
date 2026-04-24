# charmbracelet/crush#2679 — short tool descriptions by default

**What changed.** Flips the default for `CRUSH_SHORT_TOOL_DESCRIPTIONS`: the env var now defaults to *enabled*, so `FirstLineDescription` returns just the first non-empty line of the embedded markdown tool description. Setting `CRUSH_SHORT_TOOL_DESCRIPTIONS=0` restores the prior (full) behavior. The parsing predicate changes from `!v` (treat parse-error as enabled) to `err == nil && !v` (require an explicit `0`/`false`/etc. to opt out).

**Why it matters.** Tool descriptions are repeated in every system prompt; on small-context local models they can crowd out actual work. The PR claims two weeks of internal testing showing tangible quality and cost wins. Sensible default for the user mix Crush is targeting.

**Concerns.**
1. **Silent default change with model-quality implications.** Behavior of any user who never set the env var changes overnight. Models that *did* benefit from the long descriptions (e.g. specific frontier models that tool-use better with extended schemas) will degrade silently. There is no in-product log line announcing that the descriptions were truncated. Worth a one-time `slog.Debug` or release-notes callout.
2. **First-non-empty-line is a fragile contract.** A tool whose markdown begins with a callout admonition, an `# H1` title, or an empty front-matter line gets a useless first "description." `FirstLineDescription` should be more defensive — e.g. skip headings and blockquote markers, or require the doc to opt in by leading with a paragraph. With the new default this lands in every tool's schema.
3. **Parse-error semantics change.** Before: any unparseable value (e.g. `CRUSH_SHORT_TOOL_DESCRIPTIONS=yes`) was treated as "enabled." After: any unparseable value is treated as "use default" → still enabled. Same end state today, but a future flip of the default would behave differently for the same env var. Worth at least a comment.
4. **No test for the new default path.** All existing tests run under `testing.Testing()` which short-circuits the env check entirely. The behavioral change is uncovered.
5. The escape hatch is sufficient and the rollback story is one env var. Acceptable risk if (1) gets a release-note line.
