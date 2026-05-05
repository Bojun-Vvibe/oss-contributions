# openai/codex#21092 — [codex] Clean up experimental feature popup backend wiring

- Head SHA: `a0124597d7353b5ec5e886b0c1cfc2a7ea85fbc2`
- Author: canvrno-oai
- Verdict: **merge-after-nits**

## Summary

Refactors the TUI experimental-features popup to load its rows from
the existing `experimentalFeature/list` app-server endpoint and
persist toggles via the app-server config API. Aligns local and
remote (ACP) behavior, and tightens the save path to write only
changed rows. Adds optional `cwd` and `profile` to
`ExperimentalFeatureListParams` so the server can compute effective
enablement against the right config layer.

## Specific references

- `codex-rs/app-server-protocol/schema/json/ClientRequest.json:788-802`
  adds two optional, nullable string fields to
  `ExperimentalFeatureListParams`:
  ```json
  "cwd":     { "description": "Optional working directory used to resolve project config layers.", "type": ["string", "null"] },
  "profile": { "description": "Optional config profile used to compute effective feature enablement.", "type": ["string", "null"] }
  ```
- Same additions mirrored into
  `codex_app_server_protocol.schemas.json` and `.v2.schemas.json` —
  protocol surface stays consistent across v1 and v2 schema dumps.
- TUI side wires the popup to call `experimentalFeature/list` with
  the active `cwd`/`profile` and only sends a write for rows whose
  effective enablement actually changed.

## Reasoning

This is a low-risk consolidation: the popup was previously reading
config off a local-only path, which meant a remote-session (ACP)
user toggling an experimental flag would either no-op or write to
the wrong scope. Routing through the app-server gives one source of
truth for both modes.

The schema addition is fully additive (both new fields nullable +
unrequired); existing callers don't break.

Nits:

1. **`cwd` semantics on remote sessions.** When the request comes
   from an ACP client, what does "cwd" mean — the client's local
   directory, or the remote server's? The schema description
   ("Optional working directory used to resolve project config
   layers") doesn't say. Worth a sentence in the schema or in the
   handler doc.

2. **No fallback semantics documented.** If `profile` is provided
   but doesn't exist, does the server fall back to the default
   profile, error, or return rows with empty enablement? Specify.

3. **"only changed rows" is the right behavior, but make sure the
   diff includes a test** that asserts unchanged-row toggles do
   *not* trigger a config write — otherwise this optimization will
   silently regress the next time someone refactors the save path.

Solid cleanup; merge once the two doc-level questions are answered.
