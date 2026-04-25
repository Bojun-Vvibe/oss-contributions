# cline/cline #10377 — fix: expose Plan/Act mode as an accessible radio group

- **Repo**: cline/cline
- **PR**: [#10377](https://github.com/cline/cline/pull/10377)
- **Head SHA**: `5791548205d2ec3376eb16ecb74cc90a69c5bd1d`
- **Author**: toby-bridges
- **State**: OPEN (+194 / -26)
- **Verdict**: `merge-after-nits`

## Context

Issue #4932: the Plan/Act mode picker in `ChatTextArea` was rendered
as two `<div role="switch">` elements inside a click-handler container.
Screen readers reported each as a separate switch with no group
semantics, and arrow-key navigation didn't work. The picker is also
the only way for keyboard-only users to switch modes other than the
global shortcut, so this is a real accessibility gap.

## Design

Three coordinated changes in `webview-ui/src/components/chat/ChatTextArea.tsx`:

1. **`onModeSelect(targetMode)` extracted from `onModeToggle`**
   (lines 993-1019). Same proto call (`togglePlanActModeProto`) but
   parameterized on the target mode rather than always flipping. The
   early-return `if (mode === targetMode) { textAreaRef.current?.focus(); return }`
   at line 995 is correct for radio semantics — selecting the
   currently-selected radio is a no-op except for refocus. The
   existing `onModeToggle` becomes a thin wrapper at line 1023-1026
   that computes the next mode and delegates, so the `Cmd/Ctrl+Shift+P`
   shortcut path keeps working unchanged.

2. **`handleModeOptionKeyDown`** at lines 1029-1051. Implements the
   WAI-ARIA APG radio-group keyboard pattern:
   - Space / Enter → select focused option
   - Left / Up → select Plan
   - Right / Down → select Act
   
   Mapping arrows to *absolute* options ("Left always means Plan")
   rather than *relative* ("Left means previous") is the right call
   for a 2-option radio group: it's more discoverable and matches
   user intent on macOS segmented controls and the W3C
   "directional radio group" pattern. For a 3+-option group I'd ask
   for relative navigation, but here it's fine.

3. **JSX rewrite at lines 1631-1665**:
   - Container becomes `role="radiogroup"` with `aria-label="Plan or Act mode"` ✓
   - Each option becomes a `<button role="radio" type="button" aria-checked>`
     with `tabIndex={mode === m.toLowerCase() ? 0 : -1}` (roving
     tabindex — exactly right for radio groups so Tab lands on the
     selected option, not all options)
   - `border-0` added to the button class to neutralize the default
     button border styling that would otherwise visually break the
     existing slider design
   - `onFocus` / `onBlur` wired to the existing tooltip state so
     keyboard users now also see the per-option tooltip — nice touch,
     not just bare-minimum a11y

## Test coverage

New file `webview-ui/src/components/chat/__tests__/ChatTextArea.spec.tsx`
(diff shows it but truncated). From the visible portion: it stubs
`StateServiceClient.togglePlanActModeProto` with a
`vi.fn().mockResolvedValue({ value: false })`, mocks
`useExtensionState` to return `mode: "plan"`, and presumably exercises
the role/click/key paths. Per the description the test asserts roles
and selecting Act mode. Reasonable scope; would be nice to also assert
roving `tabIndex` (only one option has `tabIndex=0` at any time) and
that arrow keys actually fire the proto call.

## Risks / Nits

1. **`role="switch"` → `role="radio"` is a semantic downgrade for
   *some* assistive tech.** A switch implies binary on/off; a radio
   in a 2-option group implies "pick one of two equivalent
   alternatives." Plan vs. Act is genuinely "two alternative modes,"
   so radio is more accurate. But screen-reader users who learned
   the old "switch" announcement will hear something different now.
   Mention in CHANGELOG / release notes.

2. **Arrow-key absolute mapping is a documented design choice but
   easy to miss.** A short JSDoc comment on `handleModeOptionKeyDown`
   ("Left/Up → Plan; Right/Down → Act, regardless of currently
   focused option") would prevent a well-meaning future PR from
   "fixing" this to relative navigation.

3. **`onClick={() => onModeSelect(m.toLowerCase() as Mode)}`** — the
   `as Mode` cast is fine because `m` comes from a literal
   `["Plan", "Act"]`, but if the array ever grows you've lost
   compile-time safety. Defining
   `const MODE_OPTIONS: ReadonlyArray<{ label: string; value: Mode }> = [...]`
   as a module constant would be cleaner. Optional.

4. **`onMouseOver` retains the old "if Plan show Plan, else Act"
   logic** (line 1654) — slightly redundant ternary
   `m.toLowerCase() === "plan" ? "plan" : "act"` since `m.toLowerCase()`
   is already the right value. Could simplify to
   `setShownTooltipMode(m.toLowerCase() as Mode)`. Trivial.

5. **No screenshot or VoiceOver/NVDA test recording in PR.** A11y PRs
   benefit from an attached recording showing the screen-reader
   announcement before/after. Not required, but would help reviewers
   feel the change.

6. **Author flags they could not run the full proto generation
   pipeline due to Apple Silicon Rosetta requirement.** That's
   honest disclosure; CI on Linux runners will validate. No action.

## Verdict rationale

`merge-after-nits`. Correct ARIA pattern, correct keyboard model,
roving tabindex done right, tooltip stays consistent. The downgrade
note for release notes and the JSDoc on the keydown handler are
both housekeeping items, not blockers.

## What I learned

The "switch vs. radio" distinction matters: switch = on/off of one
thing, radio = one-of-N choice. When you migrate, plan to update
docs / screencasts / accessibility-statement language since the
announced control type changes. Roving tabindex is the
underappreciated half of accessible radio groups; without it, every
option is in the tab order and the group feels broken to keyboard
users even with correct roles.
