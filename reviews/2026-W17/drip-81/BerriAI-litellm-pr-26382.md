# BerriAI/litellm PR #26382 — Move 'Store Prompts in Spend Logs' toggle to Admin Settings

- **PR:** https://github.com/BerriAI/litellm/pull/26382
- **Author:** ryan-crabbe-berri
- **Head SHA:** `2c3c8aa4ea2c`
- **Stats:** +227 / -425 across 10 files
- **Verdict:** `merge-after-nits`

## What this changes

Relocates the "Store Prompts in Spend Logs" + "Maximum Spend Logs Retention Period" controls from a one-off modal under the Logs view (`SpendLogsSettingsModal`) into a proper Admin Settings tab card (`LoggingSettings`), wired into `AdminPanel.tsx:366-370` as a new tab key `logging-settings` between `UISettings` and `HashicorpVault`. The component file is moved (`view_logs/SpendLogsSettingsModal/` → `Settings/AdminSettings/LoggingSettings/`), the component is reshaped from a modal-with-`isVisible/onCancel/onSuccess` props to a self-contained card that renders inline, and the test file is renamed and updated accordingly.

The net diff is `-425 / +227` because the modal scaffolding (visibility prop, cancel handler, modal-close button, modal-render-when-hidden tests) goes away entirely. The remaining surface is just the form (toggle + retention input + save button), the same `useStoreRequestInSpendLogs` hook for mutation, and the same `useDeleteProxyConfigField` for clearing the retention field when left empty on submit. The behavior under save is preserved: enabled toggle + retention value → write both keys via `mutateAsync`; disabled toggle → write `store_prompts_in_spend_logs: false` and call `mockDeleteField` to remove the retention key.

## What's right

The location move is the correct call. "Store prompts in spend logs" is a privacy-and-compliance toggle that belongs in Admin Settings next to SSO, allowed IPs, and the SCIM config — not buried behind a modal launched from the Logs viewer where you'd only stumble into it after you'd already noticed that prompts weren't being captured. The card-with-inline-save shape is the same shape used by `UISettings` and `SSOSettings`, so the new tab inherits the existing submit/notification/refetch UX without re-litigating any patterns.

The test file rename plus the focused trim from 8 modal-specific assertions down to 5 card-specific assertions is the right size: dropped tests are `should render the modal` / `should render cancel and save buttons` / `should call onCancel when cancel button is clicked` / `should call onCancel when modal close button is clicked` / `should render form fields with initial values` (folded into the new "should render the card with title and form fields"). Retained tests cover the toggle, the input, the two submit branches (with/without retention), the success notification, and the delete-field path. That's the right behavioral coverage for the new shape.

The `useProxyConfig` mock got upgraded from a blanket `vi.mock(...)` to an `await vi.importActual<...>` partial-mock at lines 14-22 of the test file, preserving real exports and only stubbing the two hooks the test needs. That's a meaningful test-quality improvement that often gets forgotten on refactors, so credit due.

## Things to fix before merge

**The PR description is "Move 'Store Prompts in Spend Logs' toggle to Admin Settings" but the wider rename (component path, test path, breadcrumb in the UI tab tree) is a one-way migration that callers outside the dashboard tree need to know about.** A `git grep -r "SpendLogsSettingsModal"` should confirm no other component still imports the old name. The launch point inside the Logs viewer (the original "Settings" button that opened the modal) is presumably also being removed in this PR — confirm by grepping for the deleted import and verifying no orphan button remains pointing at a now-unrendered modal.

**The new tab key `logging-settings` (`AdminPanel.tsx:367`) doesn't appear in the deep-link / tab-route table** that lets users land on a specific admin tab via URL — verify by checking whether `AdminPanel.tsx`'s tab list is the source of truth for routing or whether there's a parallel `tabKeys` enum / route config. If the latter, this needs a matching entry there.

**Two test-helper edits at lines 127, 152 changed `expect.any(Object)` trailing-comma style** (added the trailing comma per Prettier 3.x default). That's harmless churn but mixed with semantic diff makes review harder — consider running the project's formatter as a separate commit.

**`onSuccess` callback semantics changed** — the modal version called both `mockNotificationsManager.success` and the parent's `onSuccess` (presumably to close the modal); the card version only fires the notification. Confirm there's no parent that was relying on `onSuccess` to refresh some unrelated piece of state when the toggle changed (e.g., the Logs view repaginating to reflect new retention).

**The `Logging Settings` tab label is fine but generic** — consider `Spend Log Privacy` or `Prompt Logging` to make the privacy intent visible at the tab level. This is a reasonable place to drop a one-line subtitle in the card explaining what the toggle does, since "store prompts in spend logs" reads ambiguously to a non-engineer admin.

## Bottom line

Right move (privacy-relevant toggle out of a buried modal into Admin Settings), proportional test refactor, with one test-quality upgrade folded in. Confirm the orphan-launch-point and tab-route-table checks, and ship.
