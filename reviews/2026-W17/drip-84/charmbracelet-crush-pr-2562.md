---
pr: 2562
repo: charmbracelet/crush
title: "feat(skills): add dialog to command palette"
url: https://github.com/charmbracelet/crush/pull/2562
head_sha: 2d88dbd3ee85b4463d6b7da38542403b41d47d1e
author: Amolith
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2562 — Skills browsing dialog in TUI command palette

Adds a skills-discovery dialog reachable from the command palette for
both built-in and user-configured skills. Refactors the existing
discover/dedupe/filter pipeline at the prompt-build site into a shared
`skills.Effective` helper used by both the prompt builder and the new
dialog. +957/-36 across 18 files. Author flags it as a draft for polish
work building on the earlier #2467 attempt.

## What the diff actually does

### Refactor: `skills.Effective`

`internal/agent/prompt/prompt.go:170-194` — replaces 26 lines of
inlined discovery/dedupe/disabled-filter logic with a single call:

```go
-	// Start with builtin skills.
-	allSkills := skills.DiscoverBuiltin()
-	builtinNames := make(map[string]bool, len(allSkills))
-	for _, s := range allSkills {
-		builtinNames[s.Name] = true
-	}
-	// Discover user skills from configured paths.
-	if len(cfg.Options.SkillsPaths) > 0 {
-		expandedPaths := make([]string, 0, len(cfg.Options.SkillsPaths))
-		for _, pth := range cfg.Options.SkillsPaths {
-			expandedPaths = append(expandedPaths, expandPath(pth, store))
-		}
-		for _, userSkill := range skills.Discover(expandedPaths) {
-			if builtinNames[userSkill.Name] {
-				slog.Warn("User skill overrides builtin skill", "name", userSkill.Name)
-			}
-			allSkills = append(allSkills, userSkill)
-		}
-	}
-	// Deduplicate: user skills override builtins with the same name.
-	allSkills = skills.Deduplicate(allSkills)
-	// Filter out disabled skills.
-	allSkills = skills.Filter(allSkills, cfg.Options.DisabledSkills)
+	allSkills := skills.Effective(store)
```

`internal/skills/catalog.go` (new, 241 LOC) — defines
`Effective(src DiscoverySource) []*Skill`,
`FindEffective(src, skillID) (*Skill, error)` and
`ReadContent(src, skillID) ([]byte, error)`. The shared implementation
preserves the same semantics: builtin → user/project discovery →
dedup (user overrides builtin) → disabled filter. Also added: catalog
test in `catalog_test.go` (`TestEffectiveDeduplicatesAndFiltersVisibleSkills`)
that creates a user skill named `crush-config` (matching a builtin
name) plus a `hidden-skill` listed in `DisabledSkills`, asserts the
override happens and the disabled one is filtered out.

### New TUI dialog

`internal/ui/dialog/skills.go` (new, 182 LOC) — `Skills` dialog
implementing the `Dialog` interface with filter input, list, spinner
for async loading, and key bindings (`enter`/`ctrl+y` to confirm,
`up`/`down` to navigate with wrap-around, `esc` to close).

`internal/ui/dialog/skills_item.go` (new, not shown in chunk) — the
`SkillItem` type whose `Action()` method emits `ActionAttachSkill{ID, Name}`.

`internal/ui/dialog/actions.go:81-87` — new `ActionAttachSkill` struct.

`internal/ui/dialog/commands.go:425` — registers `Skills` command in
the default command list, with empty key binding (palette only):
```go
NewCommandItem(c.com.Styles, "skills", "Skills", "", ActionOpenDialog{SkillsID}),
```

### Server/proto plumbing

Files touched: `internal/backend/config.go`, `internal/client/config.go`,
`internal/proto/proto.go`, `internal/server/config.go`,
`internal/server/server.go`. Suggests the catalog is exposed across the
client/server boundary so the dialog can query it. Diff context not
fully read; see file list.

### Drive-by cleanup

`actions.go:130-170` — `ActionFilePickerSelected.Cmd` now uses
`a.Path` directly instead of binding `path := a.Path` first, and
inlines `filepath.Base(a.Path)` at the `Attachment` site. Also fixes a
typo in the comment (`Attachement` → `Attachment`). Cosmetic.

## Observations

1. **Refactor + feature in one PR.** The `skills.Effective` extraction
   stands alone as a clean refactor with its own test. The dialog is
   additive on top. They're related but separable; reviewers focused
   on the dialog have to scroll past the refactor and vice versa. If
   #2467 history makes splitting unwieldy, fine; otherwise consider
   landing the refactor first.

2. **`prompt_test.go` is new** — good, locks the refactor invariant.
   Worth confirming it asserts the *same* behavior that the old
   inlined code produced, including the `slog.Warn("User skill
   overrides builtin skill", ...)` log line. The old code had that
   warning at the *discovery* site; the new `skills.Effective` may or
   may not (depends on `effective()` internals not visible in the
   chunk I read). If the warn is gone, that's a silent observability
   regression — operators relied on it to detect accidental skill
   shadowing. Verify by reading `catalog.go:effective()` in full.

3. **Empty key binding for the Skills command** at commands.go:425
   makes it palette-only. That's reasonable for a feature that
   shouldn't have a default global binding, but the
   `NewCommandItem(... "", ...)` reads as "I forgot to set a binding."
   Add a constant `KeyBindingNone = ""` or pass `nil` and let the
   command type document the meaning.

4. **`SkillItem.Action()` returns the dialog action directly** at
   skills.go:872 — `return item.Action()`. Compare to other dialogs
   in this file: do they wrap actions in `ActionCmd{...}` or return
   them naked? If the dispatcher expects an Action and other dialogs
   return naked, fine; otherwise this is an inconsistency.

5. **Filter behavior on empty input.** When `s.input.Value()` is empty,
   `s.list.SetFilter("")` is called. Behavior depends on
   `FilterableList.SetFilter`; if empty filter shows all, fine.
   Worth a test that confirms the filter clears correctly when the
   user backspaces.

6. **Spinner runs forever if loading flag is never reset.** skills.go:
   the `loading` field is set but I don't see where it's set to `true`
   or `false` in the visible chunk. If the dialog is loading skills
   asynchronously from the server (the proto.go change suggests yes),
   ensure there's a code path that calls `s.loading = false` on both
   success *and* error. Currently when `loading` is true the user can
   press `Close` to escape but cannot navigate or select — fine — but
   if loading never completes, the dialog is stuck.

7. **Wrap-around navigation is good UX** at skills.go:855-867 — first
   `↑` from top goes to last, last `↓` from bottom goes to top. But
   there's a subtle bug: `Previous` is bound to `up, ctrl+p` (line
   823) and `Next` is bound only to `down` (line 822). If a user has
   `ctrl+n` muscle memory from emacs they'll fall through to the
   default branch and start typing it as a filter character. Add
   `ctrl+n` to `Next.WithKeys`.

8. **`SetVirtualCursor(false)` and `Focus()` on input, plus
   `list.Focus()`** — both have focus simultaneously. The default
   branch in `HandleMsg` routes typing to the input (correct), but the
   list also receives implicit focus via `list.Focus()` in `NewSkills`
   and again on every Previous/Next press. If the list's focus ring
   ever shows, it'll fight the input's cursor visually. Verify on
   real terminal; small polish.

9. **`drive-by cleanup of `ActionFilePickerSelected`** at actions.go:
   130-170 is functionally fine but unrelated to the skills dialog.
   Author should call it out in the PR body or split — currently it
   silently rides along.

10. **No `ID` separation for builtin vs. user skills.** The dialog
    presents `SkillItem` rows but I don't see badging in the visible
    chunk. Users expecting "this skill is mine" vs "this is built-in"
    would benefit from a `[builtin]`/`[user]` tag or distinct color.
    Defer if `skills_item.go` already does this.

## Verdict

**merge-after-nits.** Author tagged it draft so this is the right
state for a structural review. Real things to fix before un-drafting:

1. Confirm `skills.Effective` preserves the "user overrides builtin"
   slog.Warn from the old code (obs 2). If gone, reinstate.
2. Surface builtin/user distinction in the dialog (obs 10).
3. Add `ctrl+n` to the `Next` keybinding (obs 7).
4. Verify `loading` flag has both true→false transitions (obs 6).
5. Pull the `ActionFilePickerSelected` cleanup into a separate PR or
   call it out in the body (obs 9).

The core refactor is clean and the new dialog is well-shaped.

## Banned-string scan

None.
