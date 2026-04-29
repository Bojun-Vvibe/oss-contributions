# openai/codex #20141 — Add ThreadManager sample crate

- **PR:** https://github.com/openai/codex/pull/20141
- **Head SHA:** `6c6ec37f10b59a0761249ba4b96b888dba34fad1`
- **Size:** +451 / −12 (17 files)

## Summary
Adds a `codex-thread-manager-sample` one-shot binary that boots a `ThreadManager` thread, submits a prompt, prints the final assistant output. To make it land cleanly, the PR also (a) renames the local helper `configured_thread_store` → public `thread_store_from_config` so the sample (and any future callers) can construct a `ThreadStore` from a `Config`, and (b) hoists the `ThreadStore` ownership from per-call (`configured_thread_store(&config)`) into `ThreadManagerState`, threading it through `ThreadManager::new` and replacing six `configured_thread_store(&config)` call-sites with `Arc::clone(&self.state.thread_store)`. Plus a new `Default for ManagedFeatures`.

## Specific observations

1. **The ownership hoist is the load-bearing change, not the sample crate.** At `core/src/thread_manager.rs:228-294` the `ThreadManagerState` struct gains `thread_store: Arc<dyn ThreadStore>`, the constructor now requires it as a positional arg, and six previously-independent call sites (`thread_manager.rs:573, 622, 648, 677, 815, 929, 962, 996`) collapse from `configured_thread_store(&config)` to `Arc::clone(&self.state.thread_store)`. This is a **behavioral change**, not just a refactor: previously each thread re-derived its store from its own (potentially differing) `config.experimental_thread_store`; now all threads in the same `ThreadManager` share whatever store was selected at `ThreadManager::new` time. That's almost certainly the right design (one process, one store), but it eliminates the ability for a fork/resume to use a different `ThreadStoreConfig` than the parent — worth a release-note callout, and worth a defensive `debug_assert!(matches!(config.experimental_thread_store, ...))` at each ex-call-site if the team wants to crash-loud on configs that try.

2. **`Default for ManagedFeatures` at `core/src/config/managed_features.rs:30-40`** is added with `/*source*/ None` — fine for the sample binary that doesn't have a config-source story, but `Default` is a public trait and now any caller can construct a `ManagedFeatures` with `None` source silently. The struct already had `pub(crate) fn from_configured(...)` as the canonical constructor. Either gate `Default` behind `#[cfg(any(test, feature = "sample"))]` or audit downstream consumers of `ManagedFeatures` to confirm `None` source is benign in their code paths (telemetry usually wants to know "where did this come from").

3. **Test-only constructor at `core/src/thread_manager.rs:352-361`** synthesizes a `RolloutConfig { codex_home, sqlite_home: codex_home, cwd, model_provider_id: "test", generate_memories: true }` and wraps it in `LocalThreadStore::new(...)`. The `cwd` falls back to `codex_home` if `std::env::current_dir()` fails (Windows / test-isolation edge). Two minor smells: (a) `sqlite_home: codex_home.clone()` collides storage and DB roots which is fine for tests but worth a comment explaining it's intentional; (b) `model_provider_id: "test".to_string()` is a string-literal magic value — if any downstream test asserts on provider id this becomes a footgun. Consider a `RolloutConfig::for_tests(codex_home)` helper.

4. **The sample binary itself (`thread-manager-sample/src/main.rs`, +323 LOC)** isn't reviewed in detail here but lives outside the workspace's normal binaries. Given it imports `codex-arg0`, `codex-login`, `codex-models-manager`, `codex-rollout`, `codex-thread-store`, `codex-utils-absolute-path` — six internal crates — it is now an implicit "is the public surface still stable enough to call from outside" smoke test. That's actually valuable, but the README at `thread-manager-sample/README.md` (+20 LOC) should explicitly say "this crate exists to keep the public ThreadManager surface honest; if you break it here, you've broken downstream consumers" so the next refactor doesn't quietly delete it for being "just a sample."

## Verdict: `merge-after-nits`

The ownership-hoist is sound and arguably overdue, but it changes behavior for the multi-config-per-process case (rare but real for `fork_thread_with_*` callers that pass a divergent config). The sample crate is a good idea. `Default for ManagedFeatures` should be scoped down or documented.

## Recommended actions
- **(Required)** Release-note the per-thread `experimental_thread_store` override no longer being respected — even one line in CHANGELOG saves a debugging trip later.
- **(Required)** Either gate `Default for ManagedFeatures` behind `#[cfg(test)]`/`#[cfg(feature = "sample")]` or add a doc comment explaining the `source: None` semantics for telemetry consumers.
- **(Nit)** Extract `RolloutConfig::for_tests(codex_home)` instead of inline-building it at `thread_manager.rs:355-361`.
- **(Nit)** README should declare the sample crate's role as a public-surface canary so a future cleanup pass doesn't delete it.
