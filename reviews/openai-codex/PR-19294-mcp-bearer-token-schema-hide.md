# PR #19294 — hide unsupported inline `bearer_token` from MCP config schema

**Repo:** openai/codex • **Author:** etraut-openai • **Status:** merged
• **Net:** +27 / −5

## Change

Adds `#[schemars(skip)]` to `RawMcpServerConfig.bearer_token` so the
generated `core/config.schema.json` no longer advertises the field.
The field stays in the deserialization struct so the runtime can
still recognize an inline `bearer_token` and produce a targeted
error pointing the user at `bearer_token_env_var`. Adds a
schema-test asserting the field is absent and `bearer_token_env_var`
is still present.

## What this is really fixing

A schema/runtime contract divergence. Editor tooling (VS Code,
Helix, Zed via `codex.toml` schema association) was telling users
`bearer_token = "..."` was valid, then the runtime refused it. The
underlying policy decision — never accept secrets inline in
`config.toml` — is correct. The schema was just lying.

## Design choice worth pausing on

There are two ways to express "field is accepted only to produce a
targeted error":

1. Keep it in `Raw…` for deserialization and skip in schema (this PR).
2. Drop it from `Raw…` entirely; let serde produce the generic
   "unknown field `bearer_token`" error, then catch that error string
   in a higher layer and rewrite it.

(1) is meaningfully better because the targeted error survives even
if the user has `additionalProperties: true` set in their schema
overlay, and because the error site is local to where validation
already happens. (2) couples the rewrite to a fragile string match
on serde's error format. The doc comment added to `RawMcpServerConfig`
should be amplified: any future "hostile field" should follow this
exact pattern and the comment should give an example.

## Sharp edge

`#[schemars(skip)]` removes the property but `additionalProperties:
false` (visible at line 1709 of the fixture) is preserved — so a
strict JSON-schema consumer will now flag `bearer_token` as an
**extra** field in the user's TOML. In `codex.toml` editors that hook
the schema for completion + diagnostics, the user will see a red
squiggle on `bearer_token = "…"` saying "Property bearer_token is
not allowed", which is *more* confusing than the runtime's current
"use bearer_token_env_var" message — the editor message gives no
hint about the right alternative.

The fix would be to add `bearer_token` to a `deprecated` or
`x-codex-rejected-with-hint` extension property on the schema, or
to add a per-field `description` via a sibling `RawMcpServerConfig`
schema overlay that says: "Inline `bearer_token` is not supported.
Use `bearer_token_env_var` to reference an env var." The current
PR is correct but it just moves the bad UX from runtime to editor.

## Concrete next step

Add a `description` for the deprecated path. One option: keep
`bearer_token` in the schema with `"deprecated": true` and a
`description` pointing at `bearer_token_env_var`. JSON Schema 2020-12
supports `deprecated`; `schemars` exposes `#[schemars(deprecated)]`.
That gives a much better editor experience than removing the
property entirely.

## Verdict

Net positive — the schema lie is gone, runtime error is preserved.
But the UX migration is incomplete: editors will surface a worse
message than the runtime did. A follow-up `deprecated + description`
pass would close the loop.

## What I learned

When you have a field that the runtime rejects, removing it from
the schema isn't always the friendliest move. A documented
"deprecated" with a hint outperforms both "schema lies" and "schema
silent". The deprecated marker is the missing third option.
