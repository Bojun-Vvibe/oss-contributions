# openai/codex PR #20812 — Use backend service-tier metadata in app-server and TUI

- URL: https://github.com/openai/codex/pull/20812
- Head SHA: `c76e96986d40734c68eea1667d0a0eb368dace33`
- Author: aibrahim-oai (Ahmed Ibrahim)

## Summary

The breaking-change variant of the service-tier rework (compare with #20823,
which is additive). Renames `Model.additionalSpeedTiers: string[]` →
`serviceTiers: ModelServiceTier[]` outright in the JSON schema, drops the
closed `ServiceTier` enum (`fast | flex`) and replaces it with an open
string commented as "Known values include `priority`, `flex`, and
`ultrafast`, but other backend-provided ids are allowed." Mirrored in the
v2 schema and `v2/ModelListResponse.json`.

## Review

Functionally this matches #20823 but **without** the deprecation runway —
the old field name disappears in the same revision. That has two
consequences worth surfacing on the PR:

1. Any client pinned to a slightly-older app-server-protocol package that
   still reads `additionalSpeedTiers` will get an empty array with no
   warning. Given the schema is a wire contract for the TUI and IDE
   integrations, the additive path in #20823 is friendlier; if the team
   prefers this PR, the changelog / release notes should explicitly call
   the rename out.
2. Loosening `ServiceTier` from a closed enum to a free-form string means
   downstream Rust deserializers that previously matched on the enum will
   now have to handle the unknown case. Worth confirming the
   `app-server-protocol` Rust types are updated in the same commit (the
   diff snippet shown is JSON-schema only) and that the TUI's match arms
   over `ServiceTier` get an explicit `_ =>` arm.

The new `ModelServiceTier` (`id`/`name`/`description`, all required) is
identical to #20823 — same caveat about `description` being mandatory.

## Verdict

`needs-discussion` — pick one of #20812 vs #20823 (additive +
`deprecated`) before merging; merging both will leave the schema with
both the old and new field for one version and then a compat break the
next, which is the worst of both worlds.
