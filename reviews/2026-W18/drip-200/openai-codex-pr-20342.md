# openai/codex#20342 — Move plugin files into core-plugins

- URL: https://github.com/openai/codex/pull/20342
- Head SHA: `3582f3ae081780b6e50fbdec1aa1f465aeea7e36`
- Size: +1001 / -301

## Summary

Continues the core-plugin extraction series (see #20309 "Move plugin manager out of core" already in INDEX) by lifting feature-resolution and tool-suggest-resolution helpers out of `codex-core` into the `codex-config` and `codex-app-server-protocol` crates. Adds a new `codex-rs/config/src/feature_read.rs` (`+221` lines) with `features_from_config_layer_stack(...)` and `resolve_tool_suggest_config_from_layer_stack(...)` so callers that already have a `ConfigLayerStack` can compute effective feature/tool-suggest state without pulling in `codex-core`'s runtime-only managed-feature constraints. `Cargo.lock` reflects new deps on `codex-analytics`, `codex-features`, `codex-tools`, and `toml_edit 0.24.0+spec-1.1.0`.

## Observations

- `feature_read.rs:60-95` — `features_from_config_layer_stack` builds two `FeatureConfigSource` records (one for global cfg, one for the resolved profile) and hands them to `Features::from_sources(...)` along with the caller-supplied `feature_overrides`. The doc comment at `:54-59` explicitly carves out the contract: "It intentionally does not apply core's runtime-only managed feature constraints; callers that already have a fully loaded core `Config` should keep using that config as the source of truth." That carve-out is the load-bearing semantic of the move — without it, callers will silently get a different effective feature set than the one `codex-core` would compute. Worth double-checking that every existing call site that switches to this API is one that was already not relying on the runtime constraints (PR description should enumerate the migrated call sites).
- `feature_read.rs:69-77` — the missing-profile path returns `io::Error::new(io::ErrorKind::NotFound, format!("config profile `{key}` not found"))`. The error kind is appropriate; the message is consistent with the other `codex-config` error sites. The default-profile branch returns `&default_config_profile` from a stack-local `ConfigProfile::default()` — fine because `Features::from_sources` borrows-once-and-flattens, but flag for the next reviewer that the lifetime here is tighter than it looks (the `&default_config_profile` would not survive an early return after a refactor that hoists the call further out).
- `feature_read.rs:128-160` — the `disabled_tools` accumulator uses a `HashSet<ToolSuggestDisabledTool>` for de-dup keyed on `.normalized()`. The empty-`layers` fallback at `:142-148` walks the snapshot from `effective_config()`, the populated-`layers` path at `:149-162` walks each layer in `LowestPrecedenceFirst` order. These two paths produce different ordering semantics for the resulting `Vec<ToolSuggestDisabledTool>` — the snapshot fallback preserves whatever order the merged TOML produced, the layer walk preserves lowest-to-highest precedence. If any consumer downstream depends on a stable order across the two paths there will be a silent behavior diff. The `#[cfg(test)]` module at `:171+` should grow a regression that pins the order on both paths.
- `Cargo.lock:2503-2530` adds `codex-analytics`, `codex-features`, `codex-tools`, and `toml_edit 0.24.0+spec-1.1.0` as new deps of (presumably) `codex-config`. The `toml_edit` addition specifically is worth a pause — `codex-config` already depends on `toml 0.9.11+spec-1.1.0`, and pulling `toml_edit` of a different `+spec-X.Y.Z` minor in alongside is the kind of thing that produces silent monomorphization-time conflicts on `toml::Value` re-exports. Verify the two crates resolve to the same `toml` underlying types or the build will surface this at the next dependency bump.

## Verdict

**merge-after-nits**

## Nits

- Add a test that pins the `disabled_tools` ordering invariant on both the empty-layers and populated-layers paths.
- PR body should enumerate the migrated call sites so reviewers can verify each one was already runtime-constraint-free.
- The `toml` / `toml_edit` co-presence in `codex-config` deps is worth a one-line comment in `Cargo.toml` explaining why both are needed at the same `+spec-1.1.0` family.
