# openai/codex PR #20708 — Add Windows sandbox readiness RPC

- URL: https://github.com/openai/codex/pull/20708
- Head SHA: `1785f42e231e5e6f9c53c393044c3c748154f0b0`
- Author: iceweasel-oai
- Verdict: **merge-after-nits**

## Summary

Adds a new `windowsSandbox/readiness` request to the app-server protocol so a client can ask the server "is the Windows Sandbox feature ready, not configured, or in need of an update?" without having to start a full setup flow. Introduces `WindowsSandboxReadiness` enum (`ready` | `notConfigured` | `updateRequired`) and `WindowsSandboxReadinessResponse { status }`. Updates the JSON schemas (v1 `ClientRequest.json`, combined `codex_app_server_protocol.schemas.json`, and v2 schemas) and the generated TypeScript bindings.

## Line-level observations

- `codex-rs/app-server-protocol/schema/json/ClientRequest.json` lines 5873–5894: the new request entry follows the existing convention exactly — `id`, `method` enum-pinned to `"windowsSandbox/readiness"`, `params: { "type": "null" }`. The `params: null` shape (rather than omitting `params`) is consistent with the other parameterless requests in this schema (e.g. `account/logout`). Good consistency.
- `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.schemas.json` lines 18139–18180: `WindowsSandboxReadiness` and `WindowsSandboxReadinessResponse` are added in the right alphabetical neighborhood. The enum uses lowerCamelCase variants (`notConfigured`, `updateRequired`) consistent with the surrounding TypeScript bindings, not snake_case — matches the existing protocol style.
- `codex-rs/app-server-protocol/schema/json/v2/WindowsSandboxReadinessResponse.json` (new file): standalone schema exists. Note the file is missing a trailing newline (per the diff `\ No newline at end of file`). Cosmetic — most repos lint for this. Worth fixing for consistency with neighboring schema files.
- `codex-rs/app-server-protocol/schema/typescript/ClientRequest.ts`: the giant union type gains `| { "method": "windowsSandbox/readiness", id: RequestId, params: undefined, }`. Note `params: undefined` (TS) vs `"type": "null"` (JSON Schema). Both are correct for "no params required" in their respective type systems but the asymmetry can confuse generators — verify the codegen step that emits this TS file maps `null` → `undefined` consistently for the other parameterless RPCs in the same file (e.g. `account/logout` is emitted as `params: undefined,` here too, so the mapping is consistent).
- The diff is purely additive — no existing request signatures change. No backward-compat concerns.
- I cannot see the *handler* implementation in this diff (only schema/binding changes). Reviewer should verify there is a corresponding server-side handler in a separate file (or a follow-up PR) and that the readiness logic actually distinguishes `notConfigured` from `updateRequired` rather than collapsing them.

## Suggestions

1. Add the trailing newline to `v2/WindowsSandboxReadinessResponse.json` to match the rest of the schema files.
2. In the PR description, link to the handler implementation (or note "handler in PR #..."), since this PR is binding-only and a reader can't tell from the diff alone whether the RPC is wired up end-to-end.
3. Document the semantic difference between `notConfigured` and `updateRequired` somewhere — at minimum a comment on the Rust enum definition. Right now a client has to guess what each status means and what remediation to suggest to the user.
4. Consider adding an integration test that round-trips a `windowsSandbox/readiness` request through the JSON-RPC layer and asserts the response decodes to the new enum — protects the schema/binding pair from drift.
