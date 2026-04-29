# block/goose#8912 — feat: make ollama host configurable in goose2

- **PR:** https://github.com/block/goose/pull/8912
- **Author:** @kalvinnchau
- **Head SHA:** `9b9ddf7d` (full: `9b9ddf7df016608131075ba4245ddddc2380d5e8`)
- **Size:** +120/-6 across 6 files

## What this does

Fixes a quiet UX bug in goose2: Ollama appeared "already connected" in the
Settings UI even when no `OLLAMA_HOST` had been set, because the backend
default host (`http://localhost:11434`) counted as configuration. The PR
splits the check across the Rust and TS layers:

1. **Backend**: new `fn ollama_host_configured(config) -> bool` in
   `crates/goose/src/providers/ollama.rs:125-127` only returns `true` when
   `OLLAMA_HOST` env var is set OR the user has saved a value:

   ```rust
   fn ollama_host_configured(config: &crate::config::Config) -> bool {
       std::env::var("OLLAMA_HOST").is_ok()
           || config.get("OLLAMA_HOST", false).is_ok()
   }
   ```

   `OllamaProvider::inventory_configured()` is then overridden to call this
   instead of inheriting the default `true` (line 269).

2. **Frontend catalog** (`ui/goose2/src/features/providers/providerCatalog.ts:163-176`):
   Ollama flips from `setupMethod: "local"` (zero-config) to
   `setupMethod: "config_fields"` with one field:

   ```ts
   {
     key: "OLLAMA_HOST",
     label: "Host",
     secret: false,
     required: true,
     placeholder: "localhost or http://localhost:11434",
     defaultValue: "http://localhost:11434",
   }
   ```

3. **New `defaultValue` channel on `ProviderField`**
   (`ui/goose2/src/shared/types/providers.ts:22`) plus
   `modelProviderHelpers.tsx:52` change:

   ```diff
   - return [field.key, currentValue?.value ?? ""];
   + return [field.key, currentValue?.value ?? field.defaultValue ?? ""];
   ```

   So when a user opens the Ollama setup form, the host field is
   pre-populated with `http://localhost:11434` rather than blank — they
   can save as-is to confirm "yes, default is fine" or edit before
   saving.

## Test coverage

Three new Rust tests in `ollama.rs:476-518`:
- `test_ollama_host_default_does_not_mark_inventory_configured` (no env,
  no saved config → not configured)
- `test_ollama_host_env_marks_inventory_configured` (env set → configured)
- `test_ollama_host_config_marks_inventory_configured` (saved config →
  configured)

Plus TS coverage in `providerCatalog.test.ts:7-21` (catalog entry shape)
and `ModelProviderRow.test.tsx:76-103` (form pre-fill + save round-trip).

That's a tidy matrix — the three Rust cases cover the actual decision
table for `inventory_configured`, and the TS test asserts the
default-value plumbing reaches the save handler with the expected
payload `{ key: "OLLAMA_HOST", value: "http://localhost:11434", isSecret: false }`.

## What I'd think harder about

1. **Behavior change for existing users.** Anyone currently running
   goose2 against a working default-host Ollama will, after this lands,
   suddenly see Ollama as **not configured**. They'll need to either set
   `OLLAMA_HOST` (no-op since it's already the default) or open Settings
   → Ollama → Save. That's a one-time UX hop but it's a regression for
   anyone who had it "just working". Consider a one-shot migration: on
   first run after upgrade, if Ollama responds at the default host AND no
   `OLLAMA_HOST` is configured, auto-write `OLLAMA_HOST=http://localhost:11434`
   to silently preserve the previous "works out of the box" experience.

2. **The `setupMethod: "local"` → `"config_fields"` flip is a wider
   behavior change than the title suggests.** It means Ollama no longer
   has the streamlined "local provider" path in the UI; it now shares
   the same setup pattern as Anthropic/OpenAI (form fields, save
   button). This is consistent with letting users override the host,
   but it loses the discoverability advantage of the `"local"`
   treatment. If `setupMethod: "local"` is what triggers the
   "we'll just try to talk to it" auto-detect flow, this PR may
   regress that auto-detect. Worth checking whether
   `local` vs `config_fields` differ in how aggressively the UI probes
   the provider on first render.

3. **`config.get("OLLAMA_HOST", false)` second arg.** I'm reading the
   second arg as `is_secret: false`. Worth a one-line comment because
   `false` here is non-obvious — at first glance it looks like a default
   value (i.e., "if missing return false"). A
   `// is_secret = false` comment would prevent the next reader from
   misreading.

4. **Env var precedence.** `std::env::var("OLLAMA_HOST").is_ok() || ...`
   means env wins, then saved config. That matches the rest of goose's
   precedence, but it's worth a quick check that the actual provider
   construction path (`OllamaProvider::from_env`, line 130 area) uses
   the *same* precedence — otherwise inventory says "configured" based
   on env but the runtime reads the saved config (or vice versa) and
   you get a UI/runtime split-brain. The diff doesn't show
   `from_env`'s body so I can't confirm.

## Verdict

**`merge-after-nits`** — solid bug fix with proper test coverage on
both layers, and the new `defaultValue` channel is a useful generic
addition. The two real concerns are (a) the silent UX regression for
existing default-host users and (b) confirming `from_env` and
`inventory_configured` agree on env-vs-config precedence.

## Nits

- Add a `// is_secret = false` comment on
  `config.get("OLLAMA_HOST", false)`.
- Consider a one-shot migration to auto-write `OLLAMA_HOST` for users
  upgrading from the prior zero-config behavior.
- Confirm `OllamaProvider::from_env` uses the same env-then-config
  precedence as `inventory_configured`.
