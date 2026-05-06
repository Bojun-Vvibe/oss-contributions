# openai/codex#21250 — add workspace_roots to thread responses

- Head SHA: (current open head; see PR)
- Author: see PR
- Link: https://github.com/openai/codex/pull/21250

## Notes
- `codex-rs/Cargo.lock:3400` adds `codex-utils-absolute-path` to `codex-core` deps and removes it from another crate (line 3915 area) — confirms a dependency move, not just an add. Verify the removed crate truly no longer needs it (otherwise build will break in feature-gated configs).
- Test fixtures in `codex-rs/analytics/src/analytics_client_tests.rs:162` and `client_tests.rs:109,127,146` all add `workspace_roots: Vec::new(),` to `ClientResponsePayload` constructions — mechanical and correct.
- `codex-rs/app-server-protocol/schema/json/ClientRequest.json` is regenerated; reviewer should diff the schema to confirm `workspace_roots` is optional or has a default, otherwise older clients break.

## Verdict
`merge-after-nits`

Mechanical schema/dep change. Confirm the JSON schema marks `workspace_roots` as defaultable (`Vec::new()`), and that the dep removed from the other crate is genuinely unused there.
