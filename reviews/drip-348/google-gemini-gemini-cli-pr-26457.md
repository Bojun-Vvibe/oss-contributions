# google-gemini/gemini-cli#26457 — fix(cli): improve mcp list UX in untrusted folders

- **Head SHA**: `e629fbe0ce46`
- **Verdict**: `merge-after-nits`

## Summary

Improves `gemini mcp list` behavior in untrusted folders. Previously stdio MCP servers showed "Disconnected" because `createTransport` threw; now the command short-circuits to a "Disabled" status with an explicit warning, and project-scoped servers that would otherwise be filtered out are surfaced (as Disabled) so users can see what's configured. Adds `LoadedSettings.getMergedSettingsAsIfTrusted()` to support this. +83/-9.

## Findings

- `packages/cli/src/commands/mcp/list.ts:179-181` — early-return `MCPServerStatus.DISABLED` when `!isTrusted` skips `testMCPConnection` entirely. Cleaner than relying on the transport throwing. Correctness: this assumes `isTrusted` is the single authority; if a future status (e.g. `BLOCKED`) is added that should win over `DISABLED`, that branch needs to come before this check. Worth a `// keep this last among status overrides` comment.
- `packages/cli/src/commands/mcp/list.ts:193-201` — fetching `getMergedSettingsAsIfTrusted()` only when both `!isTrusted` and the method exists is defensive (the optional-method check guards against partial mocks in tests). Fine, but the message at `:222-228` says *"User-level servers are also suppressed in untrusted folders to prevent accidental side-effects"* — which contradicts the code above that explicitly re-includes them (with Disabled status) for display. Clarify to *"User-level servers are listed for visibility but disabled to prevent accidental side-effects."*
- `packages/cli/src/config/settings.ts:351-358` — `getMergedSettingsAsIfTrusted` is documented as for `mcp list` specifically, but the method is on a generic `LoadedSettings` class. Either narrow the doc to *"intended for read-only diagnostic surfaces like `mcp list`"* or rename to make the read-only intent obvious (e.g. `previewMergedSettingsAsTrusted`). Right now nothing prevents another caller from using this to bypass trust checks.
- `packages/cli/src/config/settings.ts:380-385` — `computeMergedSettings(forceTrusted = false)` swaps both `isTrusted` *and* `workspace` (`forceTrusted ? this._workspaceFile : this.workspace`). The pair is correct, but the relationship is implicit — a future refactor could update one and forget the other. A short comment on why both must be swapped together would help.
- `packages/cli/src/commands/mcp/list.test.ts:447-487` — new test asserts `'project-server: /path/to/project/server  (stdio) - Disabled'` and `'user-server: https://example.com/user (http) - Disabled'`. Note the double-space before `(stdio)` vs single-space before `(http)` — that mirrors a formatting quirk in the production string. If that's intentional alignment, fine; if not, both prod and test should normalize.

## Recommendation

UX win and the code path is straightforward. Tighten the warning text to match the actual behavior, scope `getMergedSettingsAsIfTrusted` more carefully (doc or rename), and add the comment explaining the `isTrusted`/`workspace` co-swap. Then merge.
