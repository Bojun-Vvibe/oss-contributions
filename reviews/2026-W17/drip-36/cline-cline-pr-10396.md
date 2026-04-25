# cline/cline #10396 — fix: respect image support toggle for paste and drag-drop

- **Repo**: cline/cline
- **PR**: [#10396](https://github.com/cline/cline/pull/10396)
- **Head SHA**: `927ed937caf200bb318e33ab37791e661d20624c`
- **Author**: pierluigilenoci
- **State**: OPEN (+9 / -2)
- **Verdict**: `merge-as-is`

## Context

Closes cline#8635. The model-config "Supports images" toggle exists
to prevent the agent from sending images to text-only model
endpoints (e.g. local Ollama text models, certain provider
text-only deployments). The toggle was honored only on the file-
picker path; paste (`Ctrl/Cmd+V`) and drag-and-drop bypassed it
entirely. The downstream symptom is the agent firing three retries
into the provider before failing — wasted tokens, wasted wall-clock,
and a poor error message.

## Design

Single-file change in
`webview-ui/src/components/chat/ChatTextArea.tsx`:

1. **Derive `supportsImages` once** at lines 262-266 of the file
   (lines 259+ in the diff context):
   ```ts
   const supportsImages = useMemo(() => {
       const { selectedModelInfo } = normalizeApiConfiguration(apiConfiguration, mode)
       return selectedModelInfo?.supportsImages ?? false
   }, [apiConfiguration, mode])
   ```
   The `?? false` default is the right safe choice — unknown model
   capability falls closed, which is the conservative direction.
   `useMemo` keyed on `[apiConfiguration, mode]` is correct: those
   are the only inputs to `normalizeApiConfiguration`.

2. **Paste guard** at line 859 (was line 856):
   ```ts
   if (!shouldDisableFilesAndImages && supportsImages && imageItems.length > 0) {
   ```
   Adds the new condition. The `useCallback` deps array at line 910
   correctly adds `supportsImages` so React doesn't capture a stale
   closure.

3. **Drag-drop guard** at line 1268:
   ```ts
   if (shouldDisableFilesAndImages || !supportsImages || imageFiles.length === 0) {
       return
   }
   ```
   Same toggle, expressed as an early return — semantically
   equivalent to the paste guard.

## Risks

- **None substantive**. The change adds a third condition to two
  existing guards using a pre-existing helper. The deps-array
  update at line 910 is the kind of detail that often gets missed,
  and it isn't here.
- **UX nit (out of scope)**: when a user drops an image onto a
  text-only model the input is silently discarded with no toast or
  inline message. Users who hit this will be confused, but a UI
  message is a follow-up — this PR's job is to stop the wasted
  retries.

## Verdict

`merge-as-is` — small, correct, fixes a real wasted-retry bug. Ship.
