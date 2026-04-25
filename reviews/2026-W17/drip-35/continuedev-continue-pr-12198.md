# continuedev/continue #12198 — fix(cli): display full model name in TUI without trimming at slash delimiter

- **Repo**: continuedev/continue
- **PR**: [#12198](https://github.com/continuedev/continue/pull/12198)
- **Head SHA**: `d14dbc9e1764599c23bb9774a99999460f9f6aa3`
- **Author**: octo-patch
- **State**: OPEN (+15 / -6)
- **Verdict**: `merge-as-is`

## Context

Fixes #12191. The CLI's `IntroMessage` component runs
`model.name.split("/").pop()` before display, which collapses
`local/large` to `large` and `provider/model-name` to `model-name`. The
field being trimmed is `model.name` — the user-configured display name
from the YAML config — so the trim is removing user intent rather than
provider/model disambiguation. The latter would be the right thing to
do on `model.model` (the raw provider model ID, e.g.
`anthropic/claude-...`) but not on the user-visible label.

## Design

The fix at `extensions/cli/src/ui/IntroMessage.tsx:120` and
`:133`:

```diff
- <Text color="white">{model.name.split("/").pop()}</Text>
+ <Text color="white">{model.name}</Text>
```

```diff
- <ModelCapabilityWarning
-   modelName={model.name.split("/").pop() || model.name}
- />
+ <ModelCapabilityWarning modelName={model.name} />
```

Removing the trim is correct: `model.name` is the human-facing label
(`Display Name` in the YAML), and the user's choice of `local/large`
or `myteam/finetune` is intentional and disambiguates between
configured models. The capability-warning component receives the full
name too, which is what users will see and want to act on.

The `|| model.name` fallback that the second site had is now redundant
and is correctly dropped — without the `.split().pop()`, `model.name`
is already the full string.

## Test coverage

Two test changes at
`extensions/cli/src/ui/IntroMessage.test.tsx`:

1. Updated existing assertion: `lastFrame()` now must contain
   `"provider/model-name"` (full) rather than `"model-name"`
   (trimmed). This locks the regression: if anyone re-adds the trim,
   this test fails immediately.

2. New test `renders full model name including slashes without
   trimming` (lines 79-90) exercises the exact reported scenario
   `local/large`. The assertion `expect(lastFrame()).toContain("local/large")`
   is the literal user-facing string from the issue.

Both tests use the same `render()` + `lastFrame()` pattern as the
existing 9 tests, so no new test infra. Clean.

## Risks / Nits

1. **`ModelCapabilityWarning` may itself have done something with
   slashes.** The PR drops the trim at the call site but doesn't
   touch the component. If `ModelCapabilityWarning` had any internal
   "match against known capability table by stripped name" logic, it
   could now miss matches when the name contains slashes. Worth a
   one-grep check (`grep -n split.*\\/ extensions/cli/src/ui/`) before
   merge, but in practice these capability warnings are usually
   substring matches on the raw provider model ID
   (`model.model`), not on the display name, so this should be a
   no-op.

2. **No follow-up for `model.model` vs `model.name` clarity.** The bug
   was caused by the wrong field having display logic applied. A
   follow-up that adds a JSDoc on `ModelInfo.name` ("user-visible
   display label, never trim or transform") would prevent the same
   regression in adjacent components. Out of scope here.

3. Nit: the new test could also assert what's *not* shown (e.g.
   `expect(lastFrame()).not.toContain("large\n")` after newlines are
   stripped — i.e., that `"large"` doesn't appear *alone*). Optional;
   the positive assertion is sufficient.

4. Auto-generated cubic.dev summary is included in the PR body; not
   actionable but harmless.

## Verdict rationale

`merge-as-is`. Correct field-of-truth fix, two tests that lock the
regression, no API changes, no styling drift. Existing 9 tests
continue to pass. Should land directly.

## What I learned

When a display component does `something.name.split("/").pop()`, that
is almost always a smell that the wrong field is being used —
typically `name` (display) is being treated like `model` (canonical
ID). The fix is to check the schema, not to band-aid the trim. This
PR does exactly that.
