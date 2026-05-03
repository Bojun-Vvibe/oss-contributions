# Review: BerriAI/litellm PR #27073

- **Title:** feat(ui): allow setting router fallbacks on key edit
- **Author:** ryan-crabbe-berri
- **Head SHA:** `6033af3ecb58683fc1689242b6a39a0994b2a4ff`
- **Verdict:** merge-after-nits

## Summary

Adds per-key router-settings editing (e.g., `fallbacks`) to the key edit
modal in the dashboard. Touch points: extends `KeyResponse` with an
optional `router_settings` field, embeds a `RouterSettingsAccordion`
inside `KeyEditView`, and wires the local edit state into the submit
payload — sending `{}` to clear, omitting the field entirely when
untouched. Solid test coverage (4 new specs) using a mock accordion that
isolates the wiring from the real component.

## Specific-line comments

- `ui/litellm-dashboard/src/components/templates/key_edit_view.tsx:218-225`
  — the local state is initialized once from `keyData.router_settings`
  and the comment explicitly notes the editor is remounted on each
  open. Confirm `key_info_view` actually does conditionally render
  this; if a future change keys the modal differently, edits could
  leak between keys. Worth a brief assertion or a `key={keyData.token}`
  on the parent to make the contract explicit.
- `ui/litellm-dashboard/src/components/templates/key_edit_view.tsx:294-302`
  — the "send `{}` to clear" branch hinges on
  `Object.values(...).some(v => v !== null && v !== undefined && v !== "")`.
  This treats `0`, `false`, and empty arrays/objects as truthy, so a
  user setting `num_retries: 0` will *not* be cleared (which is
  correct), but a user clearing only the array fields (e.g. emptying
  `fallbacks` to `[]`) results in `[]` being truthy, so `{ fallbacks: [] }`
  is sent rather than `{}`. That may or may not be the intended clear
  semantics — backend behavior on `fallbacks: []` should be confirmed.
- `ui/litellm-dashboard/src/components/templates/key_edit_view.test.tsx:23-47`
  — the mock accordion is a clean boundary: it exposes the value plus
  a `set fallbacks` button. The "omit when untouched" test
  (`:163-188`) is the most valuable assertion in the suite — that is
  exactly the kind of regression that would silently overwrite a
  team-level setting if it ever broke.
- `ui/litellm-dashboard/src/components/key_team_helpers/key_list.tsx:101`
  — `router_settings?: Record<string, unknown> | null` is permissive;
  if the backend has a strict schema for the supported keys
  (`fallbacks`, `num_retries`, `timeout`, etc.) it would be worth
  reusing that type rather than `unknown`.

## Risks / nits

- No e2e test against the real `RouterSettingsAccordion`. The mock
  guarantees wiring but not that the real accordion's `onChange`
  contract matches.
- The `forceRender: true` on the Collapse keeps the accordion mounted
  even when collapsed — fine for state preservation, but it means any
  network calls inside `RouterSettingsAccordion` (it takes
  `accessToken`) fire on every key open. Verify it does not refetch
  global router config per modal open.

## Verdict justification

Useful UX addition with thoughtful test coverage and a clean
omit-on-untouched contract. The clear-semantics question on
`fallbacks: []` and the `Record<string, unknown>` looseness are nits,
not blockers. **merge-after-nits.**
