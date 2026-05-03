# Review: block/goose #8977 — Structured per-provider config block, non-destructive provider switching

- PR: https://github.com/block/goose/pull/8977
- Head SHA: `b392bd365ca5d04b4239ec486feeb847f4c58106`
- Author: ayourk
- Size: +875 / -34

## Summary

Closes two related issues:

- `#6692`: switching providers via `goose configure` destructively
  overwrites the singleton `GOOSE_PROVIDER` and `GOOSE_MODEL` keys,
  losing every previous provider's model selection.
- `#8373`: zero-config ACP providers (claude-acp, codex-acp, etc.)
  never get their `<name>_configured` marker set, so they never
  appear in the model switcher.

The PR introduces a structured `providers:` block in
`config.yaml` (parallel to the existing `extensions:` block) that
stores `{enabled, model, configured}` per provider. An automatic
migration converts the old flat-key layout on first load. All
existing legacy keys (`GOOSE_PROVIDER`, `GOOSE_MODEL`,
`<name>_configured`) continue to be read for backward compatibility,
and intercepts in the config-management routes redirect legacy
writes to the new block.

## Specific citations

- `crates/goose/src/config/providers.rs` (+254, new) — defines
  `ProviderEntry { enabled, model, configured }` plus
  `set_active_provider`, `get_active_provider`, `get_active_model`,
  `get_provider_entry`, `set_provider_entry`. This is the new public
  API; everything else in the diff is a call-site migration to use
  it.
- `crates/goose/src/config/migrations.rs` (+356, new) — one-time
  migration runner that reads the legacy flat keys, builds the
  `providers:` block, and writes it back. Critical to get right
  because it runs on first load against every existing user
  config.yaml. Without seeing the file directly here, the questions
  to validate before merge: idempotency (running twice must be a
  no-op), atomicity (partial migration on crash must not corrupt the
  yaml), and backup behaviour (does it leave the legacy keys in
  place?).
- `crates/goose-server/src/routes/config_management.rs:171-189` —
  the legacy-key intercept on `upsert_config`. When a client `POST`s
  `key="GOOSE_PROVIDER"` it gets routed to
  `set_active_provider(...)` instead of the raw `config.set(...)`.
  Same for `GOOSE_MODEL`. Subtle: the `GOOSE_MODEL` branch reads the
  *current* provider via `config.get_goose_provider()` and writes
  `(provider, model)`. If the client is mid-flight switching
  providers and the model write arrives before the provider write,
  the model would be associated with the *old* provider's entry.
  Likely benign but worth a minute of thought on the ordering
  contract clients should follow.
- `crates/goose-server/src/routes/config_management.rs:254-271` —
  symmetric intercept on `read_config` so a legacy reader sees the
  same string back. Also routes `"active_provider"` (a new key name)
  to the same getter, allowing new clients to use the cleaner name.
- `crates/goose-server/src/routes/config_management.rs:373-394` —
  `providers()` endpoint enriched with `saved_model: Option<String>`.
  This is what fixes the model-switcher UX from #8373 — the
  switcher can now show the previously-saved model per provider.
- `crates/goose-server/src/routes/config_management.rs:872-893` —
  `configure_provider_oauth` now writes the new
  `ProviderEntry { configured: true, ... }` instead of just setting
  a `<name>_configured` boolean. Resolves #8373 for OAuth providers.
- `crates/goose-server/src/routes/utils.rs:118-150` —
  `check_provider_configured` is the consult point. Order of checks:
  (1) new structured `providers:` block, (2) legacy
  `<name>_configured` marker for OAuth, (3) zero-config providers
  with `_configured` marker OR currently-active provider. The third
  branch is new — treating "currently active" as "configured" for
  zero-config providers fixes the bootstrap case where a user
  manually edits config.yaml.
- `crates/goose-cli/src/commands/configure.rs:845-848` and
  `crates/goose-cli/src/session/builder.rs:358-374` — the resolver
  chain now consults `goose::config::get_active_provider/_model`
  *before* the legacy flat keys, so existing users see their last
  active selection without losing per-provider model state.
- `crates/goose/src/providers/{amp_acp, claude_acp, codex_acp}.rs`
  — one-line edits each, presumably to adapt to the new
  `ProviderEntry` shape for the zero-config path.

## Verdict

**needs-discussion**

## Rationale

The design is sound and the bugs being fixed are real and
user-visible. A structured per-provider config block is the obvious
right answer to "switching A→B→A loses A's model selection," and the
parallel with `extensions:` keeps the config.yaml shape consistent.
The legacy-key intercepts and reverse-compat reads are exactly the
right hooks to make the migration invisible to existing API clients.

The reason for `needs-discussion` rather than `merge-after-nits` is
that **the migration is the load-bearing piece** and I haven't
inspected `migrations.rs` (356 lines) line-by-line. Migrations that
edit `config.yaml` in place against a heterogeneous fleet of user
configs need:

1. **Idempotency**: running the migration twice on the same input
   must produce identical output. If `providers:` already exists,
   the migration should detect that and short-circuit, not double-
   merge.
2. **Atomicity / crash safety**: write to a temp file then `rename`,
   so an interrupted migration leaves either the old config or the
   new one, never a half-written hybrid.
3. **Backup**: writing a `config.yaml.pre-providers-migration.bak`
   alongside is cheap insurance and standard practice for
   destructive auto-migrations.
4. **Legacy-key removal policy**: does the migration delete
   `GOOSE_PROVIDER`/`GOOSE_MODEL` after copying them, or leave them
   in place? Both are defensible but the choice affects what
   downgrading to a pre-migration goose binary looks like.
5. **Test coverage**: I'd want migration tests covering (a) no
   legacy keys → no-op, (b) only `GOOSE_PROVIDER` → migrated, (c)
   both → migrated, (d) `providers:` already present + legacy keys
   present → either reconcile or skip with a warning, (e) malformed
   yaml → error cleanly without overwriting.

The other discussion item is the `read_config` intercept at
`config_management.rs:254-271` — it now returns `Value::Null` when
no provider is configured instead of an error from
`config.get(...)`. That's a breaking change for any existing client
that error-handles the missing-key case rather than null-checks.
Worth calling out in the changelog.

A few smaller items that should be nits if the migration concerns
above are addressed:

- The `GOOSE_MODEL` upsert intercept at line 181 silently no-ops if
  no provider is configured (the `if let Ok(provider)` falls
  through). It should probably return an error so a client setting
  the model before the provider gets a clear signal rather than a
  200 with the value not actually persisted.
- `providers.rs` should derive `PartialEq` on `ProviderEntry` if it
  doesn't already, to make the migration tests trivially comparable.
- The `or_else(|| goose::config::get_active_provider(config))` at
  `builder.rs:362` runs before the legacy fallback. Confirm the
  ordering matches what users expect when both are populated during
  the migration window.

## Discussion items

1. Migration safety review (idempotency, atomicity, backup, legacy
   removal policy, test coverage).
2. `read_config` `Null` vs error semantic change — call out in
   release notes.
3. Tighten `GOOSE_MODEL`-without-provider intercept to return an
   error instead of silently no-op'ing.
