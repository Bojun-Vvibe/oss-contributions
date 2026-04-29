# PR #20098 — fix: ignore dangerous project-level config keys

- **Repo:** openai/codex
- **Link:** https://github.com/openai/codex/pull/20098
- **Author:** owenlin0
- **State:** OPEN
- **Head SHA:** `726716f4037fe14d62c0ea7442b4b386581511a1`
- **Files:** `codex-rs/config/src/loader/mod.rs` (+30/-1), `codex-rs/config/src/state.rs` (+4/-0), `codex-rs/core/src/config/config_loader_tests.rs` (+86/-0), more (+181/-5)

## Context

Project-local `.codex/config.toml` is loaded from repository contents — i.e., adversary-controlled when you `git clone` an untrusted repo and `cd` into it. The codex effective-config merge previously allowed a project-local TOML to override credential-routing fields like `openai_base_url`, `chatgpt_base_url`, and the `model_provider`/`model_providers` tables. That's a credential-exfiltration primitive: a malicious `.codex/config.toml` could silently route every request (and the OAuth bearer/API-key Authorization header that goes with it) to `https://attacker.example/v1` as soon as the user opens the workspace.

The blast radius is broader than just `openai_base_url` — `profile`/`profiles` can select a fully-attacker-defined provider, and `experimental_realtime_ws_base_url` covers the realtime-API code path.

## What changed

A surgical denylist applied at exactly the right layer (`codex-rs/config/src/loader/mod.rs:50-62`):

```rust
const PROJECT_LOCAL_CONFIG_DENYLIST: &[&str] = &[
    "openai_base_url",
    "chatgpt_base_url",
    "model_provider",
    "model_providers",
    "profile",
    "profiles",
    "experimental_realtime_ws_base_url",
];
```

Three structural pieces:

1. **`sanitize_project_config(config) -> (TomlValue, Vec<String>)`** at `mod.rs:719-734` — pops each denylist key from the project TOML's top-level table and records what was removed. Returning the ignored-key list (rather than just dropping silently) is the right shape; it lets the loader surface a startup warning and lets the test suite assert exactly what was filtered.

2. **`ConfigLayerEntry.ignored_project_config_keys: Vec<String>`** at `state.rs:67` — added field carrying the per-layer drop record. The `new`/`new_with_raw_toml`/`new_disabled` constructors all default it to `Vec::new()`, so non-project layers are unaffected and the diff against existing call sites stays mechanical.

3. **Wiring at `mod.rs:998-1004`** — `load_project_layers` calls `resolve_relative_paths_in_config_toml` then `sanitize_project_config` then `project_layer_entry`. Order matters: relative-path resolution runs first because the denylist keys don't carry paths; sanitization runs *before* the value enters the layer-merge pipeline so no downstream merge can resurrect the denied keys.

## Design analysis

This is the right shape.

1. **Denylist applied at the loader, not the consumer.** The alternative (filter at every consumer of `effective_config()`) would be a sprawling diff and a permanent footgun. Filtering at the layer-entry boundary means every reader of `effective_config` is automatically safe by construction. Good defense-in-depth posture.

2. **Field-level rather than table-level.** `model_providers` is a TOML table; the PR removes the *whole* table from the project layer. Correct — partial table merging would let an attacker introduce a single malicious provider entry that the user's profile selection might happen to pick up later. Whole-table removal closes that.

3. **`profile` selector is on the list.** Even with `model_providers` filtered, leaving `profile` settable from project config would let an attacker pin the user to a profile name that happens to exist in their *user* config but is the wrong one (e.g. forcing them onto a profile that disables sandboxing or has elevated permissions). Removing `profile`/`profiles` plugs this.

4. **The test (`config_loader_tests.rs:1646-1729`) covers the right surface.** Sets up a project TOML with all 7 denied keys plus a benign `model = "project-model"`, asserts `project_layer.ignored_project_config_keys` equals the full denylist, and asserts `effective_config().get("model") == Some("project-model")` — i.e., benign keys still merge through. The tail loop `for key in &project_layer.ignored_project_config_keys { assert!(project_layer.config.get(key).is_none()) }` is the structural-impossibility check that the layer's own `config` field doesn't retain the denied keys after sanitization.

## Risks

1. **Silent demotion may surprise existing project users.** Someone running with a benign-but-customized `model_provider = "azure"` in `.codex/config.toml` will now see their setting silently dropped. The PR description mentions "surface a startup warning telling users to move those settings to user-level config.toml" but the diff snippet I see only shows the layer-entry plumbing — the actual warning emission point isn't in the head excerpt. Confirming the warning fires at startup (and at what log level) is a release-note-worthy item.

2. **No `[denied]` namespace escape hatch.** Someone with a *trusted* project (e.g. internal team monorepo where the `model_providers` table is part of the desired workflow) has no way to opt back in. That's probably fine for the threat model — the trust boundary is set at "project root is adversary-controlled by default" and the user-level override exists — but worth calling out in the PR body that no opt-in/whitelist mechanism is provided. The "warning telling users to move those settings to user-level" is the prescribed migration path.

3. **Top-level only.** `sanitize_project_config` operates on `config.as_table_mut()` and only removes keys at the root. A nested `[some_other_section] openai_base_url = "..."` would slip through. Today there's no such nesting in the schema, but if the schema grows, the denylist needs to grow with it. A code comment near `PROJECT_LOCAL_CONFIG_DENYLIST` documenting the "top-level only" semantics would prevent silent drift.

4. **Companion `config_loader_state` ratchet.** The `ConfigLayerEntry::new`-family update in `state.rs` adds the new field with a `Vec::new()` default. Test struct-literal sites in `config_loader_tests.rs` are correctly updated (lines 333, 1422, 1528) but external crates that construct `ConfigLayerEntry` literally would break — worth confirming this struct is `pub(crate)` or that no external crate has reason to construct it.

## Verdict

**Verdict:** merge-after-nits

Right defense at the right layer with the right test shape. Nits before merge:
- Confirm the startup-warning emission site exists and runs at a user-visible log level (the PR body mentions it; the diff hunks shown don't surface it).
- Add a code comment at `PROJECT_LOCAL_CONFIG_DENYLIST` documenting the top-level-only semantics so future schema growth can't sneak nested overrides past the filter.
- Release-note callout for the silent behavior change for users with benign project-level `model_provider`/`profile` overrides today.

---

*Reviewed by drip-156.*
