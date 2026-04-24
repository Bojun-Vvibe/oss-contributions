# PR #2675 — command aliases, Unicode placeholders, empty tool-call input normalization

**Repo:** charmbracelet/crush • **Author:** axeprpr • **Status:**
open • **Net:** +62 / −3

## Change

Three independent fixes packaged together:

1. Add an `exit` command alias in the command palette that maps to
   the same `tea.QuitMsg{}` as the existing `quit`.
2. Widen the user-command placeholder regex from
   `\$([A-Z][A-Z0-9_]*)` to `\$([\p{L}_][\p{L}\p{N}_]*)` so
   non-ASCII identifiers (`$数据目录`, `$ДАННЫЕ_1`, `$ÅR2`) are
   recognized.
3. Normalize empty/whitespace tool-call inputs to `{}` before
   sending history back to the model. Implemented in two places:
   `internal/agent/agent.go` (`Run` loop, when reconstructing a
   `ToolCall` from a stored row) and `internal/message/content.go`
   (`Message.ToAIMessage`, when serializing for the provider).

## Load-bearing risk

The interesting one is fix (3) — the other two are mechanical.

The duplicated `normalizeToolCallInput` lives as an unexported
helper in *both* the `agent` and `message` packages, with identical
bodies. That guarantees future drift: someone fixing edge cases
(`"null"`, `"  null  "`, a single `"\u00a0"` non-breaking space)
in one will miss the other. The right home is probably
`internal/message`, with the agent-side caller importing it. As
written, the two callers also operate on slightly different inputs
— `agent.go` operates on `tc.Input` reconstructed from a stored row
(after JSON deserialization), while `message/content.go` operates
on `call.Input` mid-`ToAIMessage`. Both should converge on the
same normalization or the divergence will silently bite.

A second concern: `{}` is a *valid JSON object*, but for some tool
schemas (those requiring a non-empty argument set), it's an
**invalid** payload. The model will receive a tool-call with empty
args and may either retry the same call (loop) or surface a typed
error. The old behavior of sending `""` would also fail downstream,
just in a different shape. So this is a behavior swap, not a clear
win — it trades "model sees malformed tool-call args" for "model
sees empty-but-syntactically-valid tool-call args, which the
provider may pass through to the tool, which then errors on missing
required fields." Worth comparing real failure modes side-by-side
before assuming `{}` is the friendlier default.

The Unicode regex is correct but expands the *attack surface* of
user-supplied commands — `$<RTL-override>foo` and other Unicode
bidi/format characters now fall through `\p{L}`. Most are harmless
in command names, but they can confuse rendering in the dialog. A
follow-up `unicode.IsLetter` + reject-bidi-marks pass would be
defensive.

The `exit` alias is fine, but adding it as a *separate*
`CommandItem` instead of a true alias means the dialog now shows
both `quit` and `exit` as distinct rows — UX-wise that's clutter,
not what most users expect from "alias."

## Concrete next step

(1) Hoist `normalizeToolCallInput` to a single home (the `message`
package) and import from `agent`. (2) Add a test fixture with a
real failing tool-call-input loop captured from production logs to
prove `{}` actually breaks the loop, not just papers over it.
(3) Treat the `exit` command as an alias in the dialog data model
(one row, multiple match-strings) rather than a duplicate item.
(4) Add a unit test for the regex against bidi/format characters
to lock down what gets accepted.

## Verdict

Three independently useful fixes, but the duplicated normalizer
and the `exit`-as-duplicate-row will rot. Worth requesting a
mini-cleanup before merge.

## What I learned

When you find the same helper in two packages, the question isn't
"DRY for elegance" — it's "which package owns the *concept*."
Tool-call input normalization is a `message`-layer concern;
agent-layer is a consumer. Also: `""` → `{}` looks like a clear
fix, but it just shifts the failure mode one layer downstream.
Loop-bug fixes need the actual loop reproduction in the test
suite or the next refactor will re-break them.
