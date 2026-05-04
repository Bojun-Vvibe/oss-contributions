# google-gemini/gemini-cli PR #26404 — fix(telemetry): stop buffering events when telemetry is disabled

- **Link:** https://github.com/google-gemini/gemini-cli/pull/26404
- **Head SHA:** `a65cda0fec699bab26bf79c50799bbb3a707811b`
- **Verdict:** `merge-after-nits`

## What it does

Closes an unbounded-buffer leak in
`packages/core/src/telemetry/sdk.ts`: when telemetry was disabled in
config, `initializeTelemetry` previously bailed before flipping
`telemetryInitialized = true`, but `bufferTelemetryEvent` (called from
every `logApi*` helper across the codebase) had two branches — if
initialized, run; else push onto `telemetryBuffer`. The "else" branch
ran for the entire process lifetime, retaining closures that captured
full request/response/error events including conversation contents.
"Confirmed live to retain GBs over a long-running session with frequent
API errors" per the test-doc comment at `sdk.test.ts:367-369`.

The fix introduces a separate `telemetryDisabled` flag at `sdk.ts:106`
with three carefully-distinguished states the prior code conflated:

- **Disabled permanently** → `telemetryDisabled = true`, drop buffer, drop
  all future events. Set via `markTelemetryDisabled()` at `sdk.ts:142-148`.
- **Initialized and running** → `telemetryInitialized = true`, flush and
  forward.
- **Deferred init (waiting for OAuth)** → both flags false, keep buffering
  so events can be flushed once auth completes.

`bufferTelemetryEvent` short-circuit at `sdk.ts:128-130` checks the
disabled flag first.

Also fixes a related listener-leak: `shutdownTelemetry` was an early-return
no-op when the SDK never initialized, leaving `authListener` registered
forever on the `authEvents` emitter. The rewrite at `sdk.ts:184-237`
splits SDK-teardown (only if started) from module-state-and-listener
reset (always).

## Design analysis

- **Three-state design (`disabled` / `deferred` / `initialized`) is the
  right abstraction.** A single boolean would conflate the deferred-init
  case (where buffering is *correct* until OAuth resolves) with the
  permanently-off case (where buffering is *broken*). The two-flag
  representation lets each path do the right thing.
- **`markTelemetryDisabled` clearing the buffer at `sdk.ts:147`** is
  load-bearing — without it, events buffered before
  `initializeTelemetry` ran would persist forever even after the
  disabled state took effect. The test at `sdk.test.ts:357-372`
  specifically locks this with `beforeInitCallback` and `afterInitCallback`
  asserting both were dropped.
- **Reset at `sdk.ts:170` in init** (`telemetryDisabled = false`) handles
  the test/reload case where init is called again with telemetry
  re-enabled, without requiring an intervening `shutdownTelemetry`. The
  test at `sdk.test.ts:386-399` exercises that path.
- **`shutdownTelemetry` always resets module state** at `sdk.ts:225-237`
  even if the SDK never initialized. The `keychainAvailabilityListener`
  and `tokenStorageTypeListener` cleanup at `:241-260` was already there
  but was previously inside the early-return guard — moving it out is
  the corresponding leak fix.

## Risks / nits

1. The `initializeTelemetry` config-conflict path at `sdk.ts:217-218`
   logs an error and calls `markTelemetryDisabled()`. If the user later
   *fixes* the config conflict (e.g. flips `useCollector` off) and
   reloads, the `telemetryDisabled = false` reset at `sdk.ts:170` runs
   *before* the conflict check at `:217`, so the second init correctly
   re-enables. Correct, but worth a comment naming the recovery path
   so the field reset isn't mistakenly removed in a future cleanup.
2. `telemetryBuffer.length = 0` at `sdk.ts:227` in shutdown is the
   right primitive, but the buffer is also cleared inside
   `markTelemetryDisabled` at `:147`. Two reset sites — extract a
   single `clearTelemetryBuffer()` helper.
3. The new test at `sdk.test.ts:386-399` re-enables telemetry by
   re-mocking `getTelemetryEnabled` — important to verify the
   second `initializeTelemetry` call also goes through the
   credential-resolution path correctly, not just the disabled-flag
   reset. A follow-up test with full init (not just config reading)
   would lock the property end-to-end.
4. `markTelemetryDisabled` is called *after* the early-return at
   `sdk.ts:160` (`!getTelemetryEnabled() → markTelemetryDisabled() →
   return`). Correct ordering, but a code reader might wonder "why
   not just the return?" — a one-line comment naming the buffer-clear
   side effect would help.
5. `telemetryDisabled = false` at `sdk.ts:226` in shutdown isn't
   strictly necessary if `shutdown` is followed by process exit, but
   matters for the test scenario (init-shutdown-init-again). Worth
   a comment explaining that.

## What I learned

The "deferred initialization plus buffering" pattern is a memory-leak
trap whenever the buffering side-channel doesn't have a "stop accepting
new entries" state. The right shape is exactly what's done here: an
explicit terminal-state flag distinct from the
"not-yet-initialized-but-will-be" flag, checked *before* the buffer
push in the producer. Anything else eventually retains GBs of closures
on the long tail.
