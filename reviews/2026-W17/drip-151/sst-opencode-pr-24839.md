# sst/opencode#24839 — security: fix default-allow permission model (critical)

- **Repo:** [sst/opencode](https://github.com/sst/opencode)
- **PR:** [#24839](https://github.com/sst/opencode/pull/24839)
- **Head SHA:** `141379ac75d68b63a1c8fb0c3e82d6c56c615aa6`
- **Size:** +13 / -7 (two files: `packages/opencode/src/agent/agent.ts`, `packages/opencode/test/agent/agent.test.ts`)
- **State:** OPEN

## Context

The default agent permission shipped as `"*": "allow"`, meaning any tool not
explicitly denied — including `bash`, `write`, `edit`, `apply_patch`,
`webfetch`, `websearch`, `task` — auto-executed without confirmation. A
prompt-injection vector through any of `webfetch`-loaded content,
attacker-controlled MCP server, or poisoned dependency graph would silently
get arbitrary command execution on the user's machine.

This is the "fail-open default" anti-pattern at the riskiest possible seam.

## Design analysis

The fix at `packages/opencode/src/agent/agent.ts:84-96` flips the wildcard
to `ask` and explicitly allow-lists the read-only tools:

```ts
const defaults = Permission.fromConfig({
  "*": "ask",
  glob: "allow",
  grep: "allow",
  todowrite: "allow",
  codesearch: "allow",
  lsp: "allow",
  skill: "allow",
  doom_loop: "ask",
  external_directory: {
    "*": "ask",
```

The allow-list is well-chosen: `glob`/`grep`/`codesearch` are read-only
filesystem traversal, `lsp` is an in-process query against the language
server, `todowrite` writes to an in-memory task list (not the user's FS),
and `skill` reads from a curated skill catalog. None of these can mutate
the user's machine state or exfiltrate data via the network.

Notably absent from the allow-list (correctly): `read`. The PR description
mentions `read` should be allow-listed but the actual diff omits it. Either
the description is stale or the omission is deliberate (e.g., `read` can
exfiltrate sensitive files when paired with a later `webfetch` send).
Worth clarifying — see nit 1.

The test changes at `packages/opencode/test/agent/agent.test.ts:50-51`,
`:227`, and `:443-449` lock the new defaults across the three sites that
previously asserted the old `"allow"` shape:

```ts
expect(evalPerm(build, "edit")).toBe("ask")
expect(evalPerm(build, "bash")).toBe("ask")
// ...
expect(evalPerm(build, "webfetch")).toBe("ask")
```

The test rename `"webfetch is allowed by default" → "webfetch asks by
default"` (`:443`) is the right shape — keeps the test as a regression pin
on the new default.

## Risks / nits

1. **`read` discrepancy.** PR description claims `read` is added to the
   allow-list, but the diff at `agent.ts:85-91` does not include `read`.
   Either fix the description or add `read: "allow"` (with the caveat that
   `read` of `~/.ssh/id_rsa` followed by `webfetch` to attacker-controlled
   URL is a real exfil chain — keeping `read` at `ask` is the conservative
   call). Pick one and document the rationale in the commit message.

2. **`task` not in allow-list and not in PR-listed dangerous set.** `task`
   spawns a subagent. Under the new defaults, `task` falls through to `*`
   and prompts. That's correct, but `task` should probably be explicitly
   listed in the "dangerous tools" inventory in the PR description, since a
   subagent can chain other tools.

3. **Migration hazard for existing users.** Users with persisted agent
   configs that don't override the `*` key will silently get the new
   stricter behavior on next load. Workflows that ran unattended (CI-style
   non-interactive use of opencode) will now block on prompts that never
   come. A release note + a `--allow-all` opt-in flag for explicit
   non-interactive sessions would smooth this out. Without that, this is a
   real breaking change disguised as a security fix.

4. **Test coverage gap on the new explicit allows.** The tests assert
   `edit`/`bash`/`webfetch` now `ask`, but don't assert `glob`/`grep`/
   `codesearch`/`lsp`/`skill`/`todowrite` are still `allow`. A regression
   that flipped one of those to `ask` would break the workflow without
   failing tests. Add one positive assertion per new allow-listed tool.

5. **`apply_patch` and `write` not separately tested.** The PR description
   lists them as "dangerous" but the test file only locks `edit`/`bash`/
   `webfetch`. A minimal additional test pinning `write`/`apply_patch` at
   `ask` would cover the full advertised surface.

## Verdict

**merge-after-nits.** This is the right fix and the urgency is real (a
zero-friction RCE primitive in the default config). Block-on items are 1
(resolve `read` discrepancy in description vs diff), 3 (release note for
the breaking-default), and 4 (one-line positive assertions for the
allow-list). Nits 2 and 5 are follow-ups.

## What I learned

- "Default deny + explicit allow-list" is the right primitive at the
  permission boundary, even when it slightly hurts UX. The tools an agent
  can run are roughly the same primitives a remote attacker gets via
  prompt injection — the threat model is identical.
- When a fix flips a default that ships in the wild, the breaking-change
  question ("what does this do to users who never set this explicitly?")
  is just as load-bearing as the security argument. A `--non-interactive`
  / `--allow-all` opt-in turns the breaking change into an explicit choice.
- Test files that pin defaults are the second line of defense against
  regression — but only for the assertions actually written. The
  allow-list expansion needs paired positive assertions, not just the
  negative "this is no longer allow" ones.
