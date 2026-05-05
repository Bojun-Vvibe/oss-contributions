# google-gemini/gemini-cli #26490 — feat(mcp): auto-discover .mcp.json from project root

- **Head SHA:** `9a233d37eaca1330fc3c93611433b6e3c472878d`
- **Author:** @rushikeshsakharleofficial
- **Verdict:** `request-changes`
- **Files:** +37 / −6 across `packages/cli/src/config/settings.ts`, `packages/core/src/context/graph/render.ts`, `packages/core/src/context/tracer.ts`

## Rationale

This PR bundles two unrelated changes: (a) a new auto-discovery for `.mcp.json` at the workspace root in `packages/cli/src/config/settings.ts:806-830`, and (b) a perf/cleanup change in `packages/core/src/context/graph/render.ts:48-58` and `packages/core/src/context/tracer.ts:46-48` that gates `calculateTokenBreakdown` on `tracer.isEnabled`. Change (b) appears verbatim in PR #26489 — *the same author's own separate PR* — which means this PR is currently a strict superset and the two should be rebased so that #26489 lands first and this PR contains *only* the MCP discovery change. Bundling the two makes review harder, ties the small low-risk perf fix to the more contentious config-merging change, and creates a guaranteed merge-conflict scenario when one of the two lands first.

**On the actual MCP discovery change at `packages/cli/src/config/settings.ts:808-830`, the design has a real correctness problem:**

The merge logic at lines 821-824 reads:
```ts
workspaceResult.settings.mcpServers = {
  ...mcpJsonServers,
  ...(workspaceResult.settings.mcpServers ?? {}),
} as Settings['mcpServers'];
```

This makes the `.gemini/settings.json` `mcpServers` block **win** on key collision (because spread-after wins in object literals), which the comment at line 810 correctly claims is the intent. **Good** — that's the right precedence. But:

1. **The cast `as Settings['mcpServers']`** at line 823 is hiding a real type-safety issue: `mcpJsonServers` was declared `Record<string, unknown>` at line 819 because the JSON was just `JSON.parse`'d. We're spreading untyped data into a strongly-typed `mcpServers` field with no validation. If a user's `.mcp.json` has malformed entries (wrong shape, missing required keys), they'll silently propagate into the runtime config and blow up later in the MCP launcher with a much less actionable error. Either run the `mcpServers` block through the same validator the rest of `settings.json` goes through, or at minimum filter to only entries whose values look like objects.

2. **`if (!storage.isWorkspaceHomeDir())` at line 808** is the right guard against picking up `~/.mcp.json` and accidentally promoting a personal config into a workspace context, but the inverse case isn't handled: a user who runs `gemini` from a subdirectory of a workspace will not pick up `.mcp.json` from the project root. The PR title says "from project root" but the implementation reads `path.join(workspaceDir, '.mcp.json')` (line 809), which is the *workspace* directory, not the *project root* (which would be the nearest ancestor with a known marker like `.git`/`package.json`/`.gemini/`). Either rename the PR / docstring to "from workspace dir", or actually walk up to find the project root.

3. **The bare `try { ... } catch {}` at lines 811-829** silently swallows everything — `JSON.parse` errors, file-system errors, type errors. The comment at line 828 says "Ignore malformed or unreadable .mcp.json", but a malformed `.mcp.json` is a *user configuration error* that the user wants to know about. They put MCP servers in the file, restart Gemini, and... nothing happens, with no log line explaining why. At minimum, log a one-line warning to stderr (or whatever the project's debug-log channel is) so users have something to grep for when their `.mcp.json` is silently ignored.

4. **Compatibility framing.** The comment at line 809 says "compatible with Claude Code / MCP standard". Worth verifying whether the MCP standard actually has a `.mcp.json` discovery convention, or whether this is purely a Claude-Code convention being adopted — if it's the latter, the comment should say so. (And the test coverage for this should at least include a fixture file proving the merge precedence is what the comment claims.)

5. **No test for the new code path.** A 25-line addition to `_doLoadSettings` that mutates the most security-sensitive part of the settings (which child processes get spawned as MCP servers) needs at least one test covering: (i) `.mcp.json` is read and merged, (ii) `.gemini/settings.json` keys override `.mcp.json` keys, (iii) malformed `.mcp.json` doesn't crash and doesn't pollute settings, (iv) home-dir guard works.

**My verdict:** `request-changes` for the rebase against #26489 (must), the silent-error logging (should), and at minimum one test fixture demonstrating the precedence rule (must). The validator question (#1) is the more substantive ask but I'd accept a follow-up PR for it if there's appetite.
