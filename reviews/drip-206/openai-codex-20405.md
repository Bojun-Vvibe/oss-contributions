# openai/codex #20405 — feat: export effective session config snapshots

- Head SHA: `ce1c7870f3236a9222784c128f23dc6a9fcf3c46`
- Files: 12 across `codex-rs/config/`, `codex-rs/core/`, `codex-rs/features/`
- Size: +732 / -4

## What it does

Adds a debugging/audit hook that, when `config_snapshot_export_dir` is set in
`config.toml`, writes `<conversation_id>.config.toml` capturing the resolved
runtime configuration that the session actually ran with. The motivation is
real: today the live config is a pile of layered TOML + active profile + file-
backed instructions + feature defaults + runtime overrides, and there is no
way to see "what did this thread actually see" without re-deriving everything
from scratch.

Concretely:

- `ConfigToml` gains `permission_profile: Option<PermissionProfile>` at
  config_toml.rs:126-132 and `config_snapshot_export_dir: Option<AbsolutePathBuf>`
  at config_toml.rs:254-255 (with appropriate schema regen at
  config.schema.json:759-862 and :367 etc).
- A new module `core/src/session/config_snapshot.rs:594-625` defines
  `export_config_snapshot_if_configured(session_configuration, conversation_id)`
  which short-circuits unless the export dir is set, otherwise writes
  `toml::to_string_pretty` of the snapshot via `tokio::fs::create_dir_all` +
  `tokio::fs::write`.
- `SessionConfiguration::to_config_snapshot_toml()` (config_snapshot.rs:628)
  starts from `effective_config()`, then *overlays* live values:
  model/reasoning effort/summary/service tier, base & developer instructions,
  compact prompt, personality, approval policy + reviewer, permission profile
  (the new top-level field), web search mode (config_snapshot.rs:644-658).
- It then *strips* every source-only indirection: `profile`, `profiles`,
  `config_snapshot_export_dir` (so the dump is reproducible/idempotent),
  `model_instructions_file`, `experimental_instructions_file`,
  `experimental_compact_prompt_file`, `model_catalog_json`, `sandbox_mode`,
  `sandbox_workspace_write`, `default_permissions`, `permissions` (replaced
  by the resolved `permission_profile`), and the two experimental tool
  toggles (config_snapshot.rs:660-672).
- Wired into session start at `session/mod.rs:858`:
  `export_config_snapshot_if_configured(&session_configuration, conversation_id).await?`.

## What's right

- The "stale source vs resolved value" distinction is handled correctly:
  every nullable field that comes from layered indirection gets explicitly
  zeroed and the resolved value is written under a fresh canonical key.
- `MemoriesConfig` gaining `Serialize` (types.rs:263) is the minimal change
  needed; no behavior shift.
- The export is opt-in (None means no work) so there is no perf or disk
  footprint for users who don't enable it.

## Concerns

1. **Failure mode at session start.** Line 858 uses `?`, meaning a
   permission error or full disk on the export dir aborts session creation
   entirely. For a debugging feature, demoting this to a `tracing::warn!`
   on `Err` would be safer — losing a snapshot is preferable to losing
   a session.
2. **Conversation-id-keyed filename.** No rotation, no per-day pruning;
   long-lived users will accrete one file per session forever. Not a
   blocker but the docs/comment should call this out so users know to
   point this at a temp dir or wire their own logrotate.
3. **`original_config_do_not_use`.** The new module reaches into
   `session_configuration.original_config_do_not_use.as_ref()` (line 599).
   If the field name is meant to discourage exactly this kind of access,
   either rename or document why snapshot export is the legitimate
   exception.
4. **No test for the on-disk side.** The added test at line 735
   exercises `to_config_snapshot_toml()` in-memory; an integration test
   that sets `config_snapshot_export_dir = tempdir` and asserts the file
   appears + round-trips through `ConfigToml::parse(...)` would lock the
   tokio-fs path.

Verdict: merge-after-nits
