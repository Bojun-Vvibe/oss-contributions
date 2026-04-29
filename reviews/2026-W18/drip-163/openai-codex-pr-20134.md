# openai/codex#20134 — Sync installed remote plugin cache

- URL: https://github.com/openai/codex/pull/20134
- Head SHA: `ff75e5262673f40f0df82f4c940d3ea5129cdb1d`
- Size: +709 / −709, 9 files
- Verdict: **merge-after-nits**

## Summary

Replaces the legacy "startup marker file" remote-plugin sync with full reconciliation against `/ps/plugins/installed?includeDownloadUrls=true`: missing plugins are downloaded to `plugins/cache/<remoteMarketplace>/<pluginName>/<version>`, version drift triggers re-download, enable/disable state is preserved, and stale local entries are pruned. Sync is deduped via a `remote_plugin_sync_state {requested, in_flight}` flag and triggered both at startup and from `plugin/list`. Test surface is rewritten end-to-end against `wiremock` with gzip-bundle fixtures.

## Specific issues flagged

### 1. `run_remote_plugin_sync_loop` has unbounded retry-on-failure with no backoff

At `core/src/.../plugins.rs` (per diff lines 1015-1059), the loop is:

```rust
loop {
    { /* check + clear `requested` flag */ }
    match self.sync_plugins_from_remote(...).await {
        Ok(_) => info!(...),
        Err(err) => warn!("background remote plugin sync failed"),
    }
}
```

If `requested` is re-set during a failed sync (e.g. user keeps invoking `plugin/list`, which calls `maybe_start_remote_plugin_sync` again), the loop immediately retries with **no backoff and no error counting**. A misconfigured backend URL or 503 storm becomes a tight retry loop hammering the server and spamming `warn!` logs. Add an exponential backoff on the `Err` arm (e.g. 1s → 2s → 4s capped at 60s) and reset on `Ok`.

### 2. `requested` flag is consumed *before* the sync runs, creating a lost-wakeup window

The state-machine pattern at diff lines 1019-1027:

```rust
if !state.requested {
    state.in_flight = false;
    return;
}
state.requested = false;
```

clears `requested` at the *top* of each iteration. If a `maybe_start_remote_plugin_sync` call lands between the `state.requested = false` line and the actual `sync_plugins_from_remote(...).await` returning, the second caller sees `in_flight = true`, sets `requested = true`, and bails — but the in-flight sync was already past the read point so the second request's data won't be picked up until the *next* iteration. That's actually correct (it'll loop back and re-sync), but the comment/docstring should make this explicit because it's the load-bearing concurrency invariant.

### 3. Test rename `sync_plugins_from_remote_ignores_unknown_remote_plugins` → `sync_plugins_from_remote_keeps_existing_plugins_when_fetch_fails`

Per diff lines 1430-1431, the test name changed but it's not clear from the diff whether the *behavior* tested also changed. If the rename reflects a behavioral shift from "unknown remotes are ignored" to "fetch-failure preserves cache", that's a semantic change worth a release-note callout — pre-PR consumers might depend on the "ignore" behavior to filter out marketplace-side typos. Confirm in the PR description.

### 4. Missing test for the dedup race itself

`maybe_start_remote_plugin_sync` is the new public API for "sync if not already running, otherwise just request a re-sync". There's no test that exercises N concurrent callers and asserts exactly one `tokio::spawn` runs (the `should_spawn` boolean at diff lines 985-993 is the actual contract). A `loom`-style or simple `join_all([maybe_start_remote_plugin_sync; 16])` test that counts mock-server `/ps/plugins/installed` hits would lock in the dedup.

### 5. Lock-poisoning fallback silently uses stale state

At diff lines 988-991:

```rust
let mut state = match self.remote_plugin_sync_state.write() {
    Ok(state) => state,
    Err(err) => err.into_inner(),
};
```

`err.into_inner()` recovers from poisoning — fine for liveness — but a poisoned lock means a previous holder panicked mid-mutation, so `requested` and `in_flight` may be in any state. At minimum log `tracing::warn!("remote_plugin_sync_state lock was poisoned, recovering")` once at the recovery site so post-mortem of "sync seemed to never run after a crash" is diagnosable.

## Why merge-after-nits

Core design (state machine + spawn dedup + reconciliation) is sound and replaces a legitimately fragile marker-file mechanism. Items 1-2 are robustness, 3-5 are observability/tests. None block merge.
