# openai/codex PR #20857 — Add vanilla context mode

- Author: gustavz
- Head SHA: `04570ae600ffe3d31bef433f1c4bcb0d003c00ab`
- Verdict: **needs-discussion**

## Rationale

Plumbs a new `ContextMode` enum (`default | vanilla`) through the app-server
protocol — schema diffs at `codex-rs/app-server-protocol/schema/json/ClientRequest.json:536-540`
and the v2 schema at lines 4321-4326 add the enum, with three NewSession-
shaped request payloads each gaining an optional `contextMode` field
(`ClientRequest.json:3467-3476`, `:3881-3890`, `:4069-4078`). Schema is
internally consistent across all three locations and v1/v2. Earlier rounds
of this PR (`drip-304` `8d96b25`, `drip-310` `fc10fd1`) raised the same
core question that's still unresolved here: what does "vanilla" actually
omit (system prompt? tool registry? both?), and does the server validate
the mode for backward-compat clients? The diff shows only the wire-format
plumbing — no transport handler, no test pinning the semantics. Hold for
maintainer to define the mode contract before the field becomes a public
protocol surface.
