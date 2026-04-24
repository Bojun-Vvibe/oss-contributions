---
pr_number: 24179
repo: sst/opencode
head_sha: 5fcf4cc39a0e38093931bcada5033fbb743fa2a7
verdict: merge-after-nits
date: 2026-04-24
---

# sst/opencode#24179 — session-scoped permission bridge for external providers

**What changed.** Adds `packages/opencode/src/provider/external-permission-bridge.ts` (new, 49 lines): a `globalThis`-keyed `Map<sessionID, bridge>` that lets external (non-built-in) providers reuse OpenCode's existing `Permission.ask(...)` flow. `packages/opencode/src/session/llm.ts` registers a per-session bridge before `streamText`, wraps the stream return in `{ result, cleanup }`, and uses `Effect.acquireRelease` so the entry is removed when the stream lifecycle ends.

**Why it matters.** Built-in providers route through the existing permission service; external providers had no generic hook. The bridge is the smallest reusable surface to fix that without touching built-in paths.

**Concerns.**
1. **Symbol-keyed `globalThis` Map is process-wide.** `REGISTRY_KEY = Symbol.for("opencode.externalProviderPermissionBridge")` (line 7) means the registry is shared across every Layer instance in the same process. Fine for the current single-instance CLI, but worth a comment that re-instantiating the runtime in tests can leave stale entries — the cleanup closure on line 49 only runs through `acquireRelease`, so a panic before stream registration can leak.
2. **Cleanup races on identity.** `if (current === bridge) map.delete(sessionID)` (line 51) is the right shape — it avoids stomping on a re-registration for the same session. But there's no guard against double-register within one stream lifecycle: `registerExternalProviderPermissionBridge` overwrites silently (line 47). Either log at debug or assert no entry exists for `sessionID`.
3. **`MessageID.make` only on `messageID != null` (llm.ts line 93)** — good. But `tool.callID` is forwarded as a plain string with no branding; if `CallID` becomes branded later, this silently breaks. Tag with the brand constructor for parity.
4. **Errors collapse to `behavior: "deny"`.** Any throw from `perm.ask` becomes a deny with `error.message`. That's acceptable, but a permission timeout (user closes prompt) and a real exception look identical to the caller. Consider a third state or at least a structured `reason` field.
5. **The `streamText` return shape change is the riskier edit.** `acquireRelease` correctness depends on `cleanup` being idempotent and cheap — current impl is. Land it.

Bridge is small, additive, opt-in. Ship after a comment on the global registry lifetime.
