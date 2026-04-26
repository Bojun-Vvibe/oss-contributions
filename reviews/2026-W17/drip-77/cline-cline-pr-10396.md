---
pr: 10396
repo: cline/cline
sha: 927ed937caf2
verdict: merge-as-is
date: 2026-04-26
---

# cline/cline #10396 — fix: respect image support toggle for paste and drag-drop operations

- **URL**: https://github.com/cline/cline/pull/10396
- **Author**: pierluigilenoci
- **Head SHA**: 927ed937caf2
- **Size**: +9/-2 in 1 file (`webview-ui/src/components/chat/ChatTextArea.tsx`)

## Scope

Closes the gap where paste-image and drag-drop-image accepted attachments even when the currently selected model didn't advertise vision support. Adds a `supportsImages` derivation from `normalizeApiConfiguration(apiConfiguration, mode).selectedModelInfo?.supportsImages ?? false` (`ChatTextArea.tsx:262-266`) and gates both ingestion sites:

- Paste handler at `:859`: `if (!shouldDisableFilesAndImages && supportsImages && imageItems.length > 0)`.
- Drag-drop handler at `:1268`: `if (shouldDisableFilesAndImages || !supportsImages || imageFiles.length === 0) return`.
- `supportsImages` added to the paste-handler `useCallback` deps (`:910`).

## Specific findings

- **Symmetric fix at both sites.** Paste and drag-drop are the two image-ingestion paths in this textarea; both are now gated identically. No third path (e.g. file-picker dialog) is visible in the diff — reviewer should `grep -n "selectedImages\|setSelectedImages" webview-ui/src/components/chat/ChatTextArea.tsx` to confirm there's no fourth ingestion path silently bypassing the new gate. If file-picker also needs the gate, that's the next PR.

- **`useMemo` deps are correct**: `[apiConfiguration, mode]` covers both inputs to `normalizeApiConfiguration`. The model can change either via apiConfiguration mutation (provider switch) or via `mode` switch (different mode = different active model), and both trigger recomputation.

- **`?? false` fallback is the right default**: an unknown `selectedModelInfo` (e.g. custom provider not in the registry) gates *out* image attachment rather than letting it through. Conservative default.

- **No user-facing message** when paste/drop is silently ignored. A user who pastes an image into a textarea with a non-vision model will now see "nothing happens" with no toast/inline indicator. Acceptable for the no-op case (better than crashing or sending a broken request), but a single follow-up issue "show a toast when image attachment is dropped because the active model doesn't support vision" would close the UX loop. Not a blocker.

- **`supportsImages` added to the paste callback's deps array** (`:910`) — correct. Without this, the closure would capture the initial render's value and silently break on model switch.

- **Tests**: none added. ChatTextArea tests in cline historically exist (see `webview-ui/src/components/chat/__tests__/`) but the diff doesn't touch them. Two cheap tests would be (a) "paste image dispatches setSelectedImages when supportsImages=true", (b) "paste image is no-op when supportsImages=false". Skip-able for a 9-line bugfix, mention-worthy in review.

- **No drive-by changes** — the diff is tightly scoped to the bug. Author behavior is good.

## Risk

Trivially low. Pure gating on a derived boolean that already exists in the model-info schema. The failure mode of the *fix* is "user with vision model finds image-paste broken" — that requires `supportsImages` to be `false` for a model the user knows is vision-capable, which means a deeper bug in `normalizeApiConfiguration` and not this PR's fault.

## Nits

1. (Optional) confirm there's no third image-ingestion path (file-picker, IPC message) that also needs gating.
2. (Optional) follow-up issue for a toast/inline notice when image attachment is silently dropped.
3. (Optional) add two paste-handler tests for the gated/ungated branches.

## Verdict

**merge-as-is** — minimal, symmetric, conservative-default fix that closes a real bug (sending image content to non-vision models triggers provider-side errors that surface as opaque request failures). The 9-line diff is doing exactly what the title says.

## What I learned

When a UI component has multiple input paths to the same state (paste / drag-drop / file-picker / IPC), capability gating must be applied at every path *or* hoisted to the state mutator. The latter is cleaner ("setSelectedImages refuses non-supported attachments") but requires plumbing the capability check down to the setter. The former is the cheaper fix and is what cline did here, but it leaves a future-PR risk: a fourth ingestion path can bypass the gate without anyone noticing, because the gate doesn't live where the state is mutated. A grep audit at PR time covers it for now.
