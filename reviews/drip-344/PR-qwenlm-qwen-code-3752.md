# QwenLM/qwen-code#3752 — fix(cli): persist directory add entries

- PR ref: `QwenLM/qwen-code#3752`
- Head SHA: `5576773d3b551fd403dea0946e493ea42ac26924`
- Title: fix(cli): persist directory add entries
- Verdict: **merge-after-nits**

## Review

Real bug fix: previously `/directory add <path>` only mutated the in-memory
`WorkspaceContext` and the change evaporated on restart. This PR makes the command
write the resolved directory list back to workspace settings via
`settings.setValue(SettingScope.Workspace, 'context.includeDirectories', ...)`
(referenced throughout `packages/cli/src/ui/commands/directoryCommand.tsx` and
asserted in the tests at `directoryCommand.test.tsx:122-141`).

The really nice part is the env-var-form preservation in
`directoryCommand.test.tsx:225-247`: when the original settings file had
`includeDirectories: ["$HOME/existing-project"]`, the persisted list keeps the
`$HOME/...` literal and only appends the *resolved* form for the newly added entry.
That's the correct behaviour — without it, every `/directory add` would silently
expand all env vars in the user's settings file and lose the abstraction the user
chose. Good attention to detail.

Two nits:

1. The "should not persist directories skipped by the workspace context" test at
   `:152-170` confirms the no-op path, but the production code's branching on the
   `addDirectory` return / side-effect is implicit (the test mocks
   `addDirectory` to *not* push). It would be clearer if `addDirectory` returned a
   boolean / accepted-path and the caller inspected that explicitly, rather than
   inferring success by re-reading `getDirectories()`. Worth a follow-up refactor.
2. The "already-added" path (`:181-200`) suppresses the empty success message, which
   is good UX, but doesn't dedupe across case-insensitive filesystems on macOS /
   Windows. `path.normalize` doesn't lowercase. If a user does
   `/directory add /Users/Foo/proj` then `/directory add /users/foo/proj`, both will
   be added on macOS. Not a blocker for this PR but file an issue.

Net positive — fixes a real persistence bug and the env-var preservation alone is
worth the merge.
