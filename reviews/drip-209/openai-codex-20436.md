# openai/codex#20436 — rollout-trace: record permission profiles

- PR: https://github.com/openai/codex/pull/20436
- Head SHA: `976d8bc18aa70bb06fa9876d64f3d3bbfa418ba4`
- Author: bolinfest
- Files: 4 changed (`session/session.rs`, `tools/tool_dispatch_trace_tests.rs`,
  `rollout-trace/src/thread.rs`, `rollout-trace/src/thread_tests.rs`), +6 / −6

## Context

`ThreadStartedTraceMetadata` (in the `codex-rollout-trace` crate) is the
header attached to every rollout-trace bundle's first line; it gets
deserialized by replay/inspection tooling as the canonical record of "what
configuration produced this trace." Until this PR it carried a
`sandbox_policy: String` field projected from `session_configuration.sandbox_policy()`
via `format!("{:?}", ...)` — but the source-of-truth on the session is no
longer `SandboxPolicy`. It's `PermissionProfile`, with `SandboxPolicy` only
surviving as a legacy compatibility projection (cf. #20438, same author,
same series).

So new trace bundles were being stamped with metadata in a *lossier* shape
than the one the producer actually used. Specifically, the `Disabled` /
`Managed { .. }` / `External { .. }` distinction in `PermissionProfile`
collapses into "DangerFullAccess" / "WorkspaceWrite" / etc. on the way to
`SandboxPolicy::Debug`, which is irreversible at replay time.

## Design

A 1:1 field rename across four files, no logic change beyond the rename:

**Wire (`rollout-trace/src/thread.rs:60`).**
```rust
pub struct ThreadStartedTraceMetadata {
    pub model: String,
    pub provider_name: String,
    pub approval_policy: String,
-   pub sandbox_policy: String,
+   pub permission_profile: String,
}
```
This is a breaking change to the trace-bundle schema. (More on this below
under Risks.)

**Producer (`session/session.rs:526-529`).**
```rust
ThreadStartedTraceMetadata {
    model: ...,
    provider_name: ...,
    approval_policy: ...,
-   sandbox_policy: format!("{:?}", session_configuration.sandbox_policy()),
+   permission_profile: format!("{:?}", session_configuration.permission_profile()),
}
```
The session now seeds the metadata with the *active* `PermissionProfile`
debug-projection. This is the same string shape as before
(`format!("{:?}", ...)`), just sourced from a different field.

**Test fixtures.** Three test sites updated to seed
`permission_profile: "Disabled".to_string()` directly instead of
`sandbox_policy: "DangerFullAccess".to_string()`:
- `tools/tool_dispatch_trace_tests.rs:289`
- `rollout-trace/src/thread_tests.rs:40, 87, 204`

`Disabled` is the `PermissionProfile` variant that was historically projected
to `DangerFullAccess` at the legacy boundary, so the fixture-value swap is
equivalence-preserving for what the test is asserting (the thread-start
metadata round-trips through write+read).

## Risk analysis

**Schema break.** `ThreadStartedTraceMetadata` is the first record in every
rollout-trace bundle; renaming a serde field from `sandbox_policy` to
`permission_profile` will:
- Make new bundles unreadable by old replayers (old replayer expects
  `sandbox_policy`, finds `permission_profile`, deserialization fails on
  missing required field).
- Make old bundles unreadable by the new replayer (new replayer expects
  `permission_profile`, finds `sandbox_policy`).

The PR ships no `#[serde(alias = "sandbox_policy")]` on the new field and no
forward-compat shim. For an internal trace format that's accepted, but it
should be *intentional*. Grep for downstream readers: rollout-trace replay
inside the codex CLI itself, the `just` recipes that re-run a bundle, and
any internal eval pipeline that consumes thread-started lines from
production traces. If any of those need to read pre-PR bundles, this needs
either a serde alias or a one-time bundle-format-version bump.

**Debug-projection drift.** `PermissionProfile`'s `Debug` impl is what the
new field carries. If the variant names ever change (e.g. `Disabled` →
`Off`), every consumer of the trace metadata silently sees the new string.
This is the same fragility the previous code had with `SandboxPolicy::Debug`,
just on a different enum. If long-term replay stability matters,
`#[derive(Serialize)]` for `PermissionProfile` and storing the structured
form (or at least a `serde_json::Value`) would be more durable than the
debug-format string. Out of scope for this PR but worth filing.

**Test value choice.** Test fixtures pin `"Disabled"` everywhere. If
`PermissionProfile::Disabled`'s debug shape ever gains fields (e.g. a
`reason` annotation), the tests will silently still pass because the field
is a free-form `String` — but production traces will start carrying the new
structured shape. Pinning a non-trivial variant (e.g. `Managed { .. }`) in
at least one test would catch that regression class.

## Verdict

**`merge-after-nits`**

The PR does exactly what its description says and the code is clean. The
real concern is **schema break with no transition story** for existing
trace bundles. I'd request one of:

1. **`#[serde(alias = "sandbox_policy")]`** on `permission_profile` so old
   bundles still deserialize (with the legacy string content surfaced as the
   profile string — lossy but openable). Requires adjusting the producer to
   also write under both names during a deprecation window if symmetric
   compatibility matters.
2. **Bump a bundle-format-version field** in the same record so replayers
   can branch on it instead of relying on alias magic.
3. **Explicit changelog note** stating "rollout-trace bundles produced by
   this revision are not readable by replayers from versions ≤ X" and
   confirming there's no production replay pipeline depending on the old
   format.

Plus the optional follow-up: serialize `PermissionProfile` natively rather
than via `Debug`, so the trace metadata is structurally typed for replay.

## What I learned

`format!("{:?}", enum_value)` as a serde-friendly projection is a recurring
trap in trace/metadata code: it ships effortlessly and reads fine in dev,
but pins your wire format to whatever the `Debug` derivation happens to
produce, including any field reorderings, struct-variant additions, or
`Debug` impl overrides that future PRs might add. The right pattern when
the variant set is small and stable is `#[derive(Serialize)] enum
PermissionProfileTag { Disabled, Managed, ... }` and project to that.
