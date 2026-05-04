# sst/opencode #25673 — fix: propagate hashline tool.execute.before args and add ref params to edit schema

- SHA: `8cc7db039417c31a92699d8732f65c5ceffd1560`
- State: OPEN, +20/-2 across 1 file
- Files: `packages/opencode/src/session/prompt.ts`

## Summary

Two intertwined fixes for the third-party `@angdrew/opencode-hashline-plugin`:
(1) injects ref-based edit args (`startRef`, `content`, `operation`, `operations[]`, etc.) into the `edit` tool's JSON Schema so strict-schema models (DeepSeek) see them; (2) drops `oldString`/`newString` from `required` so models can call with `startRef+content`; (3) routes plugin-mutated args from `tool.execute.before` into `item.execute()` via a `hookOutput` wrapper so the hashline translation actually takes effect.

## Notes

- `packages/opencode/src/session/prompt.ts:419-440` — schema mutation is gated on `item.id === "edit"` *and* `schema?.properties`. Correct guard, but the mutation is in-place on the JSON-Schema object returned from `ProviderTransform.schema(...)`. If that helper memoizes/caches the schema across sessions or providers, this will leak hashline-specific properties into non-hashline contexts. Worth one defensive `schema = { ...schema, properties: { ...schema.properties } }` clone, or confirmation in the PR description that `ProviderTransform.schema` always returns a fresh object.
- `packages/opencode/src/session/prompt.ts:422-431` — adds 9 new properties (`content`, `operation`, `ref`, `startRef`, `endRef`, `fileRev`, `expectedFileHash`, `safeReapply`, `operations`) tightly coupled to one external plugin's contract. Bakes a third-party plugin's surface into core. A more neutral approach: expose a `tool.schema.before` plugin hook so hashline (or any future plugin) can mutate its own schema. Acceptable as a tactical fix if the maintainers are okay carrying the coupling; flag it.
- `packages/opencode/src/session/prompt.ts:432-435` — filters `required` to drop `oldString`/`newString`. This permanently weakens the schema for *all* edit calls, even when no hashline plugin is registered. A user who runs without hashline now gets a schema that lets the model send neither `oldString` nor `startRef`, and the failure surfaces only at Effect Schema decode time inside `item.execute`. Should be conditional on a hashline plugin actually being registered (e.g., check `plugin.has("tool.execute.before", "edit")` or expose an opt-in flag).
- `packages/opencode/src/session/prompt.ts:446-452` — the `hookOutput = { args }` + `item.execute(hookOutput.args, ctx)` change is the real bug fix and is correct. The previous code captured `args` by closure and passed the original (immutable from the hook's perspective). Other `tool.execute.before` consumers besides hashline benefit from this fix — worth calling out as the load-bearing change.
- No tests added. The PR body shows manual `opencode run` validation only. At minimum a unit test that registers a fake `tool.execute.before` mutating `output.args.foo = "bar"` and asserts `item.execute` sees `bar` would lock the contract for issue (3) without dragging hashline into the test fixture.
- Nit: lines 422-431 use very long single-line property declarations (200+ chars on the `operations` line). Repo otherwise wraps complex schema literals across multiple lines.

## Verdict

`merge-after-nits` — the args-propagation fix (lines 446-452) is unambiguously correct and useful for any plugin author. The schema mutation (lines 422-435) couples core to one third-party plugin and weakens the default schema for all users; gate on plugin registration or pull behind a `tool.schema.before` hook before merging. Add at least one regression test for the args-propagation path.
