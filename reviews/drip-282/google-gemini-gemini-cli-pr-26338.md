# google-gemini/gemini-cli PR #26338 — feat(memory): add Auto Memory inbox flow with canonical-patch contract

- Head SHA: `82fa1d20038bc35be0c3d84ae6053f7e4fb7fd66`
- Size: +4492 / -111, 26 files

## Specific refs

- `packages/core/src/services/memoryService.ts:+482/-11` — runtime safety net: snapshots active memory before/after the extraction agent runs and rolls back any direct writes to `MEMORY.md` / sibling `.md` files. Single-file guards added for `<projectRoot>/GEMINI.md` and `~/.gemini/GEMINI.md` (covers the case where the agent ignores prompt and writes outside the inbox).
- `packages/core/src/agents/skill-extraction-agent.ts:+169/-15` — extraction agent now writes unified-diff `.patch` files into `<projectMemoryDir>/.inbox/<kind>/extraction.patch` instead of mutating memory directly. Three storage tiers: `private`, `global`, `skills`.
- `packages/core/src/agents/types.ts:+8` — new `memoryInboxAccess` flag on agent definitions; paired with `runWithScopedMemoryInboxAccess` `AsyncLocalStorage` (in `local-executor.ts:+23/-14`) to grant a narrow execution-scoped exception. Only the literal canonical paths `<inboxRoot>/{private,global}/extraction.patch` are reachable, only while the agent is running.
- `packages/core/src/commands/memory.ts:+799` (new file) — inbox listing/apply/dismiss logic. Pre-filters patches whose `+++` headers escape the kind's allowed root after canonical resolution, so unsafe entries never surface in `/memory inbox`.
- `packages/cli/src/ui/components/InboxDialog.tsx:+292/-37` — UX for one consolidated entry per kind, atomic apply in lexical order with aggregated success/failure reporting, ESC restores prior selection (instead of jumping to row 0).
- `evals/auto_memory_contract.eval.ts:+489` (new) and `auto_memory_modes.eval.ts:+447` (new) — live-LLM evals pinning down (1) canonical filename, (2) incremental merge into existing extraction.patch, (3) absolute-path pointers in `MEMORY.md`, (4) `<projectRoot>/GEMINI.md` exclusion.
- `packages/core/src/services/memoryService.test.ts:+272`, `commands/memory.test.ts:+617` — heavy unit coverage of the rollback safety net and the apply/dismiss/listing path-allowlist.

## Assessment

Defensible architecture. The "review-only, nothing applied automatically" stance plus the runtime rollback safety net plus the inbox path-allowlist gives three independent layers — exactly right for a feature that lets a background agent propose modifications to durable user state.

Concerns worth resolving before merge:

1. **Patch size:** 4.6k LOC in a single PR including evals is hard to review. The `evals/` files (936 LOC) could land separately as their gating is "USUALLY_PASSES" anyway.
2. **`memoryInboxAccess` exception** (`agents/types.ts:+8`, `local-executor.ts:+23/-14`): scoped via `AsyncLocalStorage`, but any code that escapes the async context (e.g. a setTimeout, or a Promise that resolves after the `runWithScopedMemoryInboxAccess` block exits) loses the scope. Worth confirming the extraction agent never schedules deferred file writes.
3. **`isPathAllowed` denies main-agent writes to `.inbox/`** — good, but the check should also explicitly deny the extraction agent writing OUTSIDE its scoped patch path even while inboxAccess is active. The `runtime safety net` rollback catches it, but defense-in-depth at write time is cheaper.
4. Auto-bundling the `MEMORY.md` pointer when the agent forgets (described in PR body, implemented in `memoryPatchUtils.ts:+66/-2`) is convenient but means two different code paths produce `MEMORY.md` updates — surface for divergence.

Default-off + opt-in keeps blast radius contained.

verdict: needs-discussion