# sst/opencode PR #24482 — fix(opencode): agent create generates permissions field with deny instead of tools:false

- **PR:** https://github.com/sst/opencode/pull/24482
- **Author:** 21pounder
- **Head SHA:** `82944f83e60c9f6df1df6db4887cb0c5c85da39c`
- **Files:** 1 (+6 / -6)
- **Verdict:** `merge-as-is`

## What it does

Closes #23703. The `agent create` interactive wizard wrote a deprecated `tools: { bash: false, edit: false }` block to the new agent's frontmatter. The runtime permission system no longer reads `tools:`, so any restrictions selected during creation were silently ignored. The fix swaps the frontmatter key to `permissions:` and the values from `false` → `"deny"` to match the canonical permission shape.

## Specific reads

- `packages/opencode/src/cli/cmd/agent.ts:170-174` — the construction loop is renamed and re-typed:
  ```ts
  const permissions: Record<string, "deny"> = {}
  for (const tool of AVAILABLE_TOOLS) {
    if (!selectedTools.includes(tool)) {
      permissions[tool] = "deny"
    }
  }
  ```
  Identical iteration semantics — only the *key* (`permissions` vs `tools`) and *value* (`"deny"` vs `false`) change. Type narrowed from `Record<string, boolean>` to `Record<string, "deny">`, so the literal-string requirement is enforced at compile time. Good.
- `packages/opencode/src/cli/cmd/agent.ts:179-189` — the frontmatter type and the conditional emit also match. The `if (Object.keys(permissions).length > 0)` guard is preserved, so an agent created with all tools selected still gets a frontmatter without an empty `permissions: {}` block — clean.
- The PR description's manual repro (`agent create` → deselect bash/edit → confirm `permissions: bash: deny`) matches the diff one-for-one.

## Risk surface

Tiny. The fix is a key/value rename in a single function. No callers parse the old `tools:` key (that's exactly the bug — runtime ignored it), so there is no compatibility shim needed. Existing on-disk agent files with `tools:` are untouched by this change; if anyone relied on the silently-ignored field, behavior is unchanged for them (still ignored). New `agent create` invocations now produce a working permission block.

## Nits (not blocking)

1. A regression test under `test/cli/cmd/agent.test.ts` that runs `agent create` with two tools deselected and asserts the YAML contains `permissions:\n  bash: deny\n  edit: deny` would lock this in — the PR has no test coverage for the wizard's frontmatter output. Worth a follow-up.
2. Consider adding a one-shot migration helper (or doc note) for existing agent files generated before this fix; users with `tools: bash: false` will still have no enforcement until they re-run `agent create` or hand-edit. A `agent doctor` style scan could surface stale frontmatter. Not blocking on this PR.

Verdict: merge as-is — single-file, surgical fix for a silently-ignored field, type system pinned to the only valid value, manual repro confirmed.
