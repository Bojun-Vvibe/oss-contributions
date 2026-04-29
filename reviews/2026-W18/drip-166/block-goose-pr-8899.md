# block/goose #8899 — perf: parallelize provider resolution and eagerly init SQLite pool

- **PR:** https://github.com/block/goose/pull/8899
- **Head SHA:** `2ed8c7d459044fe31d63cfb5b7680307ad4ec42b`
- **Size:** +18 / −5 across two files (`crates/goose/src/acp/server.rs`, `crates/goose/src/providers/inventory/mod.rs`)

## Summary
Two-track perf change for goose2 startup. **Track 1** at `crates/goose/src/acp/server.rs:850-857` spawns a `tokio::spawn` background task at agent construction time that touches `session_manager.storage().pool().await` so the SQLite pool's `connect_lazy_with`-deferred ~1.6s schema check overlaps with provider resolution rather than blocking the first session-create. **Track 2** at `crates/goose/src/providers/inventory/mod.rs:233-242` rewrites `ProviderInventoryService::entries` from sequential `for provider_id in ids { entry_for_provider(...).await? }` to fan-out via `tokio::spawn` per provider + `futures::future::join_all`. Both tracks reduce wall-clock startup time at the cost of small additional concurrency complexity.

## Specific observations

1. **The eager-init at `acp/server.rs:850-857`:**
   ```rust
   // Eagerly initialize the SQLite pool so it's ready when providers/sessions need it.
   // The pool uses connect_lazy_with, so initialization only happens on first use.
   // By spawning this now, the ~1.6s schema check overlaps with provider resolution.
   let storage_clone = session_manager.storage().clone();
   tokio::spawn(async move {
       let _ = storage_clone.pool().await;
   });
   ```
   The `let _ =` discards both the result and the error. **If `pool().await` fails** (DB file unwritable, schema migration broken), the failure is silently swallowed at agent-construction time — the failure surfaces only later when an actual SQLite call hits the same error. Acceptable trade-off for the perf win on the happy path, but a `tracing::warn!("eager SQLite pool init failed: {err}; will surface on first DB use")` would preserve diagnostics without changing behavior.

2. **The fan-out at `providers/inventory/mod.rs:233-242`:**
   ```rust
   let handles: Vec<_> = ids.into_iter().map(|id| {
       let this = self.clone();
       tokio::spawn(async move { this.entry_for_provider(&id).await })
   }).collect();
   let entries: Vec<_> = futures::future::join_all(handles).await
       .into_iter()
       .filter_map(|r| r.ok().and_then(|r| r.ok()).flatten())
       .collect();
   ```
   The error-collapsing chain `.ok().and_then(|r| r.ok()).flatten()` is doing three things at once: (a) `JoinHandle::Result<T, JoinError>` → `Option<T>` (drops `JoinError`), (b) `Result<Option<Entry>, anyhow::Error>` → `Option<Option<Entry>>` (drops the per-provider error), (c) `Option<Option<Entry>>` → `Option<Entry>` (the final `flatten()`). Net: any `JoinError` (panic in the spawned task) **or** any `entry_for_provider` `Err` is silently dropped. Pre-PR behavior was different — the original `.await?` propagated `entry_for_provider` errors to the caller, so a single misconfigured provider made the whole `entries()` call fail. Post-PR: misconfigured providers silently disappear from the result.

3. **`self.clone()` per-provider in the spawn loop** — the `Clone` impl on `ProviderInventoryService` should be cheap (likely `Arc`-wrapped state), but if it's not (e.g. if it clones a `BTreeMap` of provider configs), the fan-out adds quadratic-ish cost. Worth confirming `ProviderInventoryService` is `#[derive(Clone)]` over `Arc<Inner>` and not over deeply-cloned fields. The previous serial loop only borrowed `&self`, so this is a real shape change.

4. **`tokio::spawn` vs `futures::future::join_all` of bare futures** — using `tokio::spawn` here means each provider resolution runs on a dedicated task and benefits from work-stealing across the runtime's worker pool. If this code runs inside a `current_thread` runtime (some test contexts do), `tokio::spawn` won't actually parallelize — it'll just interleave. `futures::future::join_all(ids.into_iter().map(|id| async move { ... }))` would interleave on every runtime including current_thread, and without the `JoinError`-dropping issue. Worth considering.

5. **Behavioral semantics change: pre-PR, errors propagated; post-PR, errors are silently filtered.** This is the most material change in the PR and is not called out in the title or commit message. A caller that previously got `Err(...)` for a misconfigured provider now gets `Ok(vec![...without that provider...])`. UI surfaces that displayed an error toast on missing provider will silently show a shorter list. **This is a behavioral regression dressed up as a perf optimization.**

## Risks

- **Silent error swallowing in `entries()`** — see observation #5. Pre-PR a single broken provider failed `entries()`; post-PR the broken provider just disappears. Callers that conditioned downstream behavior on "provider X is in the inventory" will silently take the no-provider branch.
- **Silent failure in eager pool init** — see observation #1. A startup that would have failed loudly (e.g. read-only DB file) now fails silently at construction and noisily on first use, far removed from the original cause.
- **`self.clone()` cost** — needs verification that `ProviderInventoryService: Clone` is `Arc`-cheap. If not, the fan-out is a perf pessimization for installations with many providers.
- **Test-runtime parallelism gotcha** — `tokio::spawn` under `current_thread` runtime doesn't parallelize. Any tests that expected the parallel speedup to materialize will see no improvement.
- **Spawned tasks are not awaited at agent shutdown** — the `tokio::spawn` at `acp/server.rs:854` creates a detached task. If the agent is dropped before the schema check completes, the task leaks until completion. Almost certainly fine (the task will finish on its own), but worth knowing.

## Suggestions

- **(Recommended)** Replace `tokio::spawn(...)` in `entries()` with `futures::future::join_all(ids.into_iter().map(|id| async move { ... }))` to interleave on every runtime context (including `current_thread`) and to keep error semantics inline (no `JoinError` channel).
- **(Recommended)** Decide what to do with per-provider `Err`: either propagate (if `entries()` should fail when any provider fails — the original behavior) or `tracing::warn!("provider {id} entry resolution failed: {err}; skipping")` and then filter (if filtering is intended). The current silent `.filter_map` is the worst of both worlds.
- **(Recommended)** Add `tracing::warn!` to the eager pool init: `if let Err(err) = storage_clone.pool().await { tracing::warn!("eager SQLite pool init failed: {err}; will retry on first use"); }`.
- **(Recommended)** Add a microbench or wall-clock log line so this change can be measured rather than asserted. Pattern: `let start = Instant::now(); ... tracing::info!("entries() took {:?}", start.elapsed());`.
- **(Optional)** Confirm `ProviderInventoryService` is `Arc<Inner>` cheaply-cloneable; if not, switch to `Arc<Self>` clone.
- **(Optional)** Consider splitting this PR — track 1 (eager pool init) is uncontroversial; track 2 (fan-out with silent error swallowing) deserves its own PR with the error-semantics decision called out explicitly.

## Verdict: `request-changes`

The eager-pool-init track is fine with a `tracing::warn!` added. The fan-out track silently changes error semantics from "fail-fast on any provider error" to "filter out failing providers" — this is a behavioral regression that is not called out in the PR title, commit message, or comment. Either the new behavior is intentional (in which case it needs a release note and explicit log/warn on filtered-out providers) or it's accidental (in which case the error path needs to propagate). As-shipped, the silent filtering will cause hard-to-debug "where did my provider go" reports. Plus the `tokio::spawn`-vs-`join_all` runtime-context concern is worth addressing before this lands.

## What I learned

When converting a `for x in xs { f(x).await? }` loop to a fan-out, the first decision is the error-propagation contract: do you want `Err` to short-circuit the whole result (use `try_join_all`), do you want partial success with errors logged (use `join_all` + explicit warn-and-filter), or do you want partial success with errors silently dropped (use `join_all` + `.filter_map(.ok())`)? Each is valid; choosing one silently while the title says "perf" is the bug class. Always state the error-semantics choice explicitly in the PR description.
