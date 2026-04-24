# anomalyco/opencode PR #24210 — feat(opencode): add /context command

- **Repo:** anomalyco/opencode
- **PR:** [#24210](https://github.com/anomalyco/opencode/pull/24210)
- **Head SHA:** `abfd8805187deda3ba186b470f3f66fd5d8632fc`
- **Author:** MartinWie
- **Size:** +898 / −35 across 15 files (2 commits)
- **Reviewer:** Bojun (drip-25)

## Summary

Adds a native `/context` slash command + `session_context` keybind
that opens a TUI dialog showing what's actually filling the
current session's context window. The dialog has two distinct
layers:

1. **Authoritative section (top)** — pulls the same numbers the
   overflow check uses (`current/usable tokens`, context limit,
   reserved output budget, in/out/reasoning/cache-read/cache-write
   from the last assistant turn, plus `OVERFLOW — compaction
   pending` and `OVER BUDGET — auto-compaction disabled` flags).
2. **Estimated breakdown (scrollable)** — per-section sizes for
   system prompt, skills, rules/memory (AGENTS.md etc.), agent,
   built-in tools, MCP tools, per-tool MCP entries, messages, and
   last user turn. Per-item numbers are explicitly labelled as
   estimates because they use a `chars/4` heuristic (not every
   slice has a real token count).

The well-designed bit: the breakdown helpers got extracted into a
new shared `session/assemble.ts` module that *both* `prompt.ts`
(the actual prompt assembler) and `context.ts` (the breakdown
service) call. So the per-section numbers can't drift from what
actually gets sent to the model.

## Key changes

### New shared assembly module: `packages/opencode/src/session/assemble.ts` (+174)

This is the architectural payoff of the PR. Previously the
system-prompt / tool-section / MCP-section rendering was
duplicated between `session/prompt.ts` and `session/llm.ts`. Both
now call through `assemble.ts`, which means the displayed
breakdown and the actual model input share the same code path.

The `prompt.ts` and `llm.ts` deltas confirm this is a true
extraction, not a parallel implementation:
- `session/prompt.ts`: +8 / −13 (net deletion, calling into
  shared module)
- `session/llm.ts`: +7 / −11 (same shape)

### New context service: `packages/opencode/src/session/context.ts` (+313)

Produces a `ContextInfo` struct with both the authoritative usage
shape and the per-section estimate breakdown. New HTTP route
`GET /:sessionID/context` exposed via
`packages/opencode/src/server/routes/instance/session.ts` (+43).
SDK regenerated (`packages/sdk/js/src/v2/gen/sdk.gen.ts +38`,
`types.gen.ts +76`).

### Overflow math single-source: `session/overflow.ts` (+11 / −3)

A small `currentTokens(...)` helper used by both the overflow
detection path and the new context service. This is the right
thing — without it, the `current` shown in the dialog can drift
from the value that triggers compaction, and you get the
"the dialog says 95% but compaction hasn't fired" footgun.

### TUI dialog component: `packages/opencode/src/cli/cmd/tui/component/dialog-session-context.tsx` (+184)

New dialog with the colored bar (green < 70%, yellow 70–90%,
red ≥ 90%/overflow), scrollable per-section breakdown.

### Dialog framework addition: `packages/opencode/src/cli/cmd/tui/ui/dialog.tsx` (+9 / −4)

`Dialog` gains a new `"full"` size tier (near-full terminal
width, smaller top padding). The plugin `Dialog` type union in
`packages/plugin/src/tui.ts` (+3 / −3) is widened to match.
Existing `medium/large/xlarge` behaviour is preserved.

### Test fixture update: `test/fixture/tui-plugin.ts` (+1 / −1)

Bumped to the widened size union so the plugin API typechecks.

### Followup commit: `fix(opencode): remove duplicate 'full' union member in Dialog size`

The first commit had `"full"` listed twice in the type union; the
fixup commit dedupes it. Worth squashing on merge so the main
commit has a clean union.

## Concerns

1. **`chars/4` heuristic is a known-lossy estimator.**
   The PR is upfront that per-section numbers are estimates. But
   the heuristic is famously bad for non-English text (CJK
   tokenises ~1 token per char, code with lots of `_` or numeric
   literals tokenises differently from prose). If a user with a
   Chinese system prompt sees the dialog say "system prompt: 800
   chars / 200 tokens" and the actual tokeniser cost is 800
   tokens, they'll think they have headroom they don't have.
   Worth either (a) using the model's actual tokeniser when one
   is loadable for the session's provider, or (b) labelling the
   per-section numbers as "approx (assumes English)" so the
   miscalibration is visible. The authoritative top section
   would still be correct, but the breakdown is the part users
   will *act* on.

2. **No persistence of the dialog open/closed state across sessions.**
   Every time the user opens a new session and wants to inspect
   context, they have to re-open `/context`. Fine as a v1, but
   worth a follow-up: a `context.show_on_overflow_warning` config
   flag that auto-opens the dialog when the bar would be red,
   so users get the diagnostic surface exactly when they need
   it without remembering the slash command.

3. **The new `assemble.ts` extraction is the load-bearing change.**
   The dialog itself is a TUI feature. The architectural value
   is the shared assembly module — but that means the breakdown
   accuracy is now coupled to whether `assemble.ts` stays in
   sync with what the model actually sees. If a future provider
   adapter does post-`assemble.ts` rewriting (e.g. adds
   provider-specific system prompt suffixes in the LLM client),
   the dialog will lie. A regression test that asserts
   "what `context.ts` reports == what gets serialized to the
   model request" — even just for one provider — would catch
   future drift.

4. **313-line `context.ts` is doing a lot for a v1.**
   Building both the authoritative usage shape and the
   estimate-breakdown sections + per-section heuristic + HTTP
   route handler in 313 lines is dense. Worth a follow-up
   split: `ContextInfo` type and authoritative computation in
   one file, breakdown estimator in another, route handler in
   a third. Easier to swap the breakdown estimator
   (heuristic → real tokeniser) without churning the route.

## Verdict

`merge-after-nits` — the architectural shape is right (shared
assembly module so the breakdown can't drift from the prompt
sent to the model is exactly the move). Two pre-merge asks: a
clear "approx (assumes English)" label on per-section
estimates, and squashing the duplicate-`"full"`-union fixup
into the main commit. Two post-merge follow-ups: real
tokeniser when available, and a regression test pinning the
"breakdown == sent-to-model" contract. The `currentTokens(...)`
helper landing in `overflow.ts` is exactly the right defensive
move and worth calling out as a model for future "two
consumers compute the same number, one of them silently
diverges" footguns.

## What I learned

The single-source-of-truth pattern for "compute a number for
display" + "compute the same number for a runtime decision"
is a common bug shape — and the right fix is almost always
to extract the computation into a shared helper that both
sites call, *not* to add a comment that says "keep these in
sync". The display-vs-decision drift is the kind of bug
users only catch by hitting the wall (compaction fires and
the dialog still says they had headroom) and then it
destroys trust in the diagnostic. The opencode team's
choice to make `currentTokens(...)` shared between
overflow.ts and context.ts is the correct shape and worth
imitating any time a UI surfaces a number that also gates a
runtime decision.
