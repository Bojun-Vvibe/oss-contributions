# PR #8886 — fix: copy and content improvements in goose2

- **Repo:** block/goose
- **Link:** https://github.com/block/goose/pull/8886
- **Author:** delkc
- **State:** DRAFT
- **Head SHA:** `75a454258b9d527594f77965594561e032357fce`
- **Files:** ~22 files across `ui/goose2/src/features/{agents,chat,projects,settings,sidebar}/...` plus locale JSON (en + es). +145/-106.

## Context

Goose2 is the next-gen UI shell. It's accumulated a few months of organic copy choices — capitalization drift, error messages that say "Unable to..." in some places and "Couldn't..." in others, a literal JSON syntax error in `sessions.json`, and a small contract mismatch where the ACP backend returns `"New Chat"` (title-case) for empty sessions while the UI's own default is `"New chat"` (sentence-case). This is a sweep PR that normalizes the lot in one pass.

## What changed

Tour of the substantive changes:

### 1. ACP title normalization (the only behavior change)

`ui/goose2/src/features/chat/lib/sessionTitle.ts:1-26`:

```ts
export const DEFAULT_CHAT_TITLE = "New chat";   // was "New Chat"

// The goose ACP backend uses "New Chat" (title case) as its default — normalize to ours.
export function normalizeAcpTitle(
  title: string | null | undefined,
): string | undefined {
  if (!title) return undefined;
  return title === "New Chat" ? DEFAULT_CHAT_TITLE : title;
}
```

Wired in at `ui/goose2/src/features/chat/stores/chatSessionStore.ts:170`:

```ts
title:
  overlay?.userSetTitle ??
  normalizeAcpTitle(session.title) ??
  overlay?.lastKnownTitle ??
  "Untitled",
```

The single-source-of-truth normalizer means every code path that ingests an ACP session title (right now just `mergeAcpSessionWithOverlay`, but trivially extendable) goes through the same normalization. This is the right shape for a contract-mismatch fix — it puts the translation at the boundary, not at every consumer.

### 2. Delete-confirmation modal restructuring

Three sites — `AgentsView.tsx:302-307`, `ProjectsView.tsx:303-308`, and an additional one in `SettingsModal.tsx:378-384` (the diff cuts off but the pattern is clear) — now interpolate the item name into the modal *title* (`Delete "My Agent" permanently?`) instead of the body. The `t("view.deleteTitle", { name: ... })` calls require the i18n string to accept a `{{name}}` interpolation slot — i.e., the locale JSON files (`en.json`, `es.json`) must be updated to match. The PR description confirms locale updates are included.

### 3. Doctor "Run fix" modal

`DoctorCheckRow.tsx:124-148`: previously rendered `check.fixCommand` inside the `AlertDialogDescription` with `font-mono` styling. Now the description carries the human-readable explanation (`runFixDescription`) and the command goes into a separate `<code className="block break-all rounded bg-muted px-3 py-2 font-mono text-xs">` block. The `<Button variant="outline" size="sm">` props on the confirm button are also removed — buttons in this dialog now use whatever default prop set the design system specifies. Worth confirming that wasn't a *deliberate* small/outline choice for visual hierarchy.

### 4. Provider/agent-name display polish

`ChatInput.tsx:341-343`:

```ts
const agentDisplayName = (
  activePersona?.displayName ?? providerDisplayName
).replace(/ \(Default\)$/, "");
```

A regex-trim that strips a trailing `" (Default)"` from agent-display labels. This is the kind of "silent string mutation in a render path" that's easy to forget about — works fine for `en` but the literal `" (Default)"` won't match localized strings. If `displayName` is ever localized (e.g. `(Por defecto)` in `es`), this trim will silently no-op. Worth a comment explaining the assumption that `displayName` arrives from a code path that only ever appends the English `" (Default)"` tag.

### 5. Section-header cleanups

`AppearanceSettings.tsx`, `DoctorSettings.tsx`, `ProvidersSettings.tsx`: all three comment-out (rather than delete) the `<p>` description block under the `<h3>` title, leaving:

```tsx
{/* <p className="mt-1 text-sm text-muted-foreground">
  {t("appearance.description")}
</p> */}
```

This is a smell. **Don't ship commented-out code** — if the description is being removed, delete it; if it might come back, leave it active and update the copy. Three commented-out blocks across three files is exactly the kind of "drift seed" that produces a confused reader six months from now wondering whether the description is hidden by intent or by accident. Also, the `t("...description")` keys probably remain in the locale JSON files unused, which a strict linter will eventually flag.

### 6. JSON syntax-error fix in `sessions.json`

The PR description mentions a missing-comma syntax error fix in EN + ES `sessions.json`. Not visible in the head excerpt of the diff but a real correctness fix — broken locale JSON would have produced a runtime parse error in the locale loader.

## Design analysis

The good:

- `normalizeAcpTitle` is the right shape: single normalizer at the boundary, called from one place currently, trivially callable from any future ingestion point.
- Delete-modal restructuring (item name in title, consequence in body) is a known UX best practice — names in titles are what the user scans first.
- Doctor `<code>` block is clearly more legible than `<AlertDialogDescription>`-as-monospace.

The mixed:

- The locale-coupling is correct *if* both `en.json` and `es.json` get the matching `{{name}}` interpolation slots in `view.deleteTitle`. Worth verifying both locale files were updated for *all three* delete sites (agents, projects, plus the third in SettingsModal). A missing locale key would fall back to the literal key string, which is ugly but not broken.
- The PR scope is wide (~22 files). It's a copy-pass, so individual file review is fast, but the total diff means a single regression could land masked by surrounding noise. The fact that this is DRAFT and "Generated with Claude Code" is in the body suggests the author already knows there's polish to do.

The bad:

- Three `{/* commented-out <p> */}` blocks. Pick one direction (delete the lines, or keep them and update the copy) — the third option is the worst.
- `replace(/ \(Default\)$/, "")` is an English-only string mutation in what's otherwise an i18n-aware codebase.

## Risks

1. **Locale-key/code drift.** Every renamed/restructured i18n key needs matching updates in *every* locale JSON (`en`, `es`, plus any others in the repo). The PR description says EN + ES are covered. The risk: any third locale not mentioned would silently fall back to key strings.

2. **Removed `Button variant="outline" size="sm"` in Doctor modal.** Was that a deliberate visual-hierarchy choice for the destructive-action confirm button? If yes, removing it means the button now gets the default (probably solid + default-size) styling, which may be visually too prominent for a confirm-fix-command action. Cheap to verify in a screenshot.

3. **Commented-out section descriptions.** As noted, this is dead-code-as-comments. Either delete or restore — don't ship the third state.

4. **English-only `(Default)` regex strip.** Will silently stop working if `displayName` is ever localized.

5. **DRAFT state.** PR is marked DRAFT and the body acknowledges it's a sweep. Don't merge until the author confirms locale parity, the commented-out blocks are resolved, and the Doctor modal button styling is intentional.

## Verdict

**Verdict:** request-changes

`normalizeAcpTitle` is a clean fix, the delete-modal restructuring is good UX hygiene, and the JSON syntax-error fix is a genuine bug. But three things should change before merge:

1. **Pick a direction for the three commented-out section descriptions.** Delete or restore — don't leave them as comments.
2. **Locale-parity audit.** Confirm every i18n key edit (especially `view.deleteTitle` with the new `{{name}}` interpolation) lands in *every* locale JSON in the repo, not just `en` and `es`.
3. **Confirm `Button` styling change in Doctor modal is intentional.** If yes, leave it; if no, restore `variant="outline" size="sm"`.

Optional but worth considering:
- Replace the English-only `replace(/ \(Default\)$/, "")` in `ChatInput.tsx` with a structural fix at the source that produced the `" (Default)"` suffix in the first place.
- Take the PR out of DRAFT after the above.

---

*Reviewed by drip-156.*
