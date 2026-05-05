# charmbracelet/crush #2809 — fix(ui): allow oauth modals to consume enter

- URL: https://github.com/charmbracelet/crush/pull/2809
- Head SHA: `61c109eaedfa26cf6b34e72099ecb8624924d8c6`
- Diff: +1 / -1, single file `internal/ui/model/ui.go` (line 1624)
- Fixes #2806

## Findings

1. The change at `internal/ui/model/ui.go:1624` is a one-line guard. Before: `if m.isAgentBusy() { return util.ReportWarn("Agent is busy, please wait...") }`. After: `if m.isAgentBusy() && !m.dialog.ContainsDialog(dialog.OAuthID) { ... }`. Located inside `handleSelectModel(msg dialog.ActionSelectModel)`, this means: when the user is on the OAuth modal selecting a model/provider that requires auth, pressing Enter no longer gets swallowed by the "agent busy" guard — the OAuth dialog gets to consume the keypress and proceed with authentication.
2. The fix is minimal and exactly matches the bug class (modal can't consume Enter because outer handler bails first). The `ContainsDialog` API call mirrors how other parts of the dialog stack do conditional handling, so the idiom is consistent.
3. **Concern**: the guard is keyed on `dialog.OAuthID` specifically. If there are other modal dialogs that also need to consume Enter while the agent is busy (e.g., a future "credit refill" modal, MCP install prompt, etc.), each will need its own carve-out. A more general fix would invert the check — only enforce "agent busy" if **no** modal is currently active — but that's a larger refactor and outside this PR's scope. The targeted fix here is the right call for a `fix:` PR; just worth noting the pattern will accumulate.
4. No tests added. UI-state tests for Bubble Tea models are notoriously awkward (most are integration via `teatest`), so the absence is consistent with project conventions. A `teatest` harness verifying that pressing Enter on an OAuth modal while a long-running tool call is in flight actually completes auth would be the ideal regression guard, but its absence isn't a blocker.
5. The downgrade path is safe: if `ContainsDialog` returns false (no OAuth modal), behaviour is identical to before. If the OAuth modal isn't in the dialog stack at all (e.g., the OAuthID constant gets renamed), `ContainsDialog` returns false and we degrade to the old behaviour gracefully.

## Verdict

**Verdict:** merge-as-is
