# openai/codex PR #20708 — Add Windows sandbox readiness RPC

- **Head SHA**: `1785f42e231e5e6f9c53c393044c3c748154f0b0`
- **Scope**: app-server protocol schemas (JSON + TypeScript) + handler

## Summary

Introduces a `windowsSandbox/readiness` request returning a `WindowsSandboxReadinessResponse { status: "ready" | "notConfigured" | "updateRequired" }`. Schema additions appear in:

- `codex-rs/app-server-protocol/schema/json/ClientRequest.json:5873-5895`
- `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.schemas.json:1580` and `:18159` (enum + response)
- `codex-rs/app-server-protocol/schema/json/v2/WindowsSandboxReadinessResponse.json` (new)
- `codex-rs/app-server-protocol/schema/typescript/ClientRequest.ts` (single line union add)

## Comments

- Enum values use camelCase (`notConfigured`, `updateRequired`) which matches the rest of the v2 schema convention — consistent.
- `params: null` (and `undefined` in the TS union) is the right call for a pure status query; no params to validate.
- New `WindowsSandboxReadinessResponse.json` is missing a trailing newline (visible in the diff hunk). Trivial.
- The diff does not show the actual handler implementation in this hunk view — assuming it's wired in `windows_sandbox` module elsewhere; reviewer should verify the dispatcher arm is added before merging.
- Three-state enum is good (better than `bool ready`) — leaves room for `updateRequired` UX flow without a future protocol break.

## Verdict

`merge-after-nits` — add trailing newline to `WindowsSandboxReadinessResponse.json` and confirm dispatcher wiring is included in the full PR (not visible in head of diff).
