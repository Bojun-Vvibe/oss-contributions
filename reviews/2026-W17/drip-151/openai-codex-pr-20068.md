# openai/codex#20068 — app-server: disable remote control without sqlite

- **Repo:** [openai/codex](https://github.com/openai/codex)
- **PR:** [#20068](https://github.com/openai/codex/pull/20068)
- **Head SHA:** `8ff72d0e5ee62a456d2f61ca52ce1cfb17b16a33`
- **Size:** +150 / -36 (5 files under `codex-rs/app-server/`)
- **State:** OPEN

## Context

Remote control persists enrollment identity (account/server/environment IDs)
in the SQLite state DB. Before this PR, if the state DB failed to open at
startup the error was silently swallowed (`.ok()`) and remote control kept
running, but every enrollment load/store became a no-op that returned
"unavailable" via `info!` log lines. The result: a process that *appeared*
remote-controlled but couldn't actually persist or recover its identity
across restarts — a classic "fail-open + log a warning" anti-pattern at a
trust boundary.

## Design analysis

Three coupled changes turn this into fail-loud:

**1. Capture and report the init error**
(`codex-rs/app-server/src/lib.rs:564-594`):

```rust
let state_db_result = codex_state::StateRuntime::init(
    config.sqlite_home.clone(),
    config.model_provider_id.clone(),
)
.await;
let state_db_init_error = state_db_result.as_ref().err().map(ToString::to_string);
let state_db = state_db_result.ok();
// ...
if let Some(err) = &state_db_init_error {
    error!("failed to initialize sqlite state db: {err}");
}
```

The error string is captured separately from the `.ok()` conversion so it
can be reported at `error!` level (vs the previous silent drop). Good
shape — keeps the existing `Option<StateRuntime>` downstream plumbing
intact.

**2. Gate `remote_control_enabled` on `state_db.is_some()`**
(`lib.rs:637-651`):

```rust
let remote_control_config_enabled = config.features.enabled(Feature::RemoteControl);
let remote_control_enabled = remote_control_config_enabled && state_db.is_some();
if remote_control_config_enabled && state_db.is_none() {
    error!("remote control disabled because sqlite state db is unavailable");
}
```

Distinguishes "feature was never enabled" from "feature was enabled but
substrate is missing." The startup-failure error message at `:644-648`
also tells the user *why* there's no transport (`"remote control disabled
because sqlite state db is unavailable"`), which is exactly the
diagnostic the previous silent drop was missing.

**3. Make the load/store helpers return `Result` instead of `Option`**
(`enroll.rs:38-51`, `:99-111`):

The previous shape `-> Option<RemoteControlEnrollment>` conflated "no
enrollment exists yet" (legitimate `None`) with "DB is unavailable so we
can't tell" (also `None` with a log line). The new shape returns
`io::Error::new(ErrorKind::NotFound, ...)` for the
"DB-unavailable" case and reserves `Ok(None)` for "no enrollment in DB."

This is the right contract narrowing — callers can now distinguish the
two failure modes and decide independently. The test sites at `:329-411`
correctly thread `.expect("first enrollment should load")` through the
new `Result` return.

**4. Runtime gate in `RemoteControlHandle::set_enabled`** (visible in
`mod.rs:1-50`): an `enable(true)` call later in process lifetime is now
rejected if the state DB was unavailable at startup, preventing a
late-bound enable from creating a half-functional remote control session.

## Risks / nits

1. **Order of `error!` calls.** `state_db_init_error` is logged at `:592`,
   then `remote_control_enabled` is computed at `:640`, then *another*
   `error!` fires at `:644` for the same root cause. Two error lines for
   one underlying issue is mildly noisy. Consider folding into one
   `error!` block at `:592` that includes the "remote control will be
   disabled" suffix when applicable.

2. **`io::Error::new(ErrorKind::NotFound, ...)` overload.** The new error
   uses `ErrorKind::NotFound` for both "enrollment not in DB" semantics
   and "DB unavailable" semantics. A custom error type
   (`StateDbUnavailable`) propagated through the call chain would let
   higher layers distinguish without string-matching the message. Not
   blocking — `NotFound` plus the descriptive string is fine for
   logging — but worth noting if anyone ever wants to take programmatic
   action on it.

3. **Test for `direct websocket connection without state db`.** The PR
   body claims this is covered. Looking at `tests.rs:325-411` the new
   assertions confirm `expect("first enrollment should load")` and
   `expect("missing account should load")` patterns, but I don't see an
   explicit test that exercises the **direct websocket path** with
   `state_db = None` and asserts the websocket connection fails. Worth
   adding a one-line `assert!(matches!(err.kind(), ErrorKind::NotFound))`
   on that path so the contract change is locked.

4. **Migration hazard.** Users running with corrupt or permissions-blocked
   `~/.codex/state.db` who relied on remote control "appearing to work"
   will now get a hard failure on startup. The error message at
   `lib.rs:644-648` is descriptive enough that they can fix it (delete
   stale DB, fix permissions), but a release note flagging the behavior
   change would help.

## Verdict

**merge-as-is.** Five-file diff, all changes are tightly scoped to the
"convert silent fail-open into loud fail-closed" theme. The
`Option → Result` contract narrowing in `enroll.rs` is the right
narrowing. Nits 1 and 3 are non-blocking polish. Notable that the test
suite was updated alongside the contract change (vs left to silently
propagate `Option`).

## What I learned

- "Fail-open + log warn" at a trust boundary is almost always the wrong
  default. Logging is invisible until something breaks; the system that's
  supposed to be operating with a substrate is now operating without one.
  Fail-loud at startup forces the user to make an explicit choice.
- The `Option<T>` → `Result<Option<T>>` refactor is the right move
  whenever a `None` was load-bearing for *two distinct* causes ("no data"
  and "can't tell"). Once the contract distinguishes them, callers can
  pick policy per case.
- A startup error message that names *both* the missing substrate and the
  feature being disabled (`"remote control disabled because sqlite state
  db is unavailable"`) is dramatically more actionable than two separate
  lines. The diagnostic-coupling cost is worth the user-experience win.
