# ollama/ollama#15716 — server: fix download stall watchdog not firing when no bytes arrive

- **PR:** https://github.com/ollama/ollama/pull/15716
- **Head SHA:** `3b754dde9b1700cfd04cb2e46ec7f63eb4adba62`
- **Files:** `server/download.go` (+12/-5), `server/download_test.go` (+63/-0, new file).
- **Verdict:** **merge-as-is**

## Context

Refs `ollama#1736` ("99% hang on `ollama pull`"). Maintainer-described
symptom: *"certain parts stall completely and zero data is received
from the backend. The connection itself is still healthy so it doesn't
trigger a retry."* Real-world consequence: the user's terminal sits at
99% with no progress for hours and the only recovery is `Ctrl-C` →
`ollama pull` again, which restarts the affected part.

## Problem — root cause analysis

The per-part stall watchdog in `server/download.go` runs as a
goroutine inside the `g.Go(...)` block in `downloadChunk`. Pre-PR
shape:

```go
g.Go(func() error {
    ticker := time.NewTicker(time.Second)
    for {
        select {
        case <-ticker.C:
            part.lastUpdatedMu.Lock()
            lastUpdated := part.lastUpdated
            part.lastUpdatedMu.Unlock()

            if !lastUpdated.IsZero() && time.Since(lastUpdated) > 30*time.Second {
                slog.Info(...)
                // reset last updated
                part.lastUpdatedMu.Lock()
                part.lastUpdated = time.Time{}
                part.lastUpdatedMu.Unlock()
                return errPartStalled
            }
        case <-ctx.Done():
            return nil
        }
    }
})
```

The PR's root-cause analysis correctly identifies a two-stage failure
mode that I'll restate:

1. **Bug A — `IsZero()` guard suppresses pre-first-byte stalls.** The
   `if !lastUpdated.IsZero() && ...` clause means the stall timer
   only ticks if the reader goroutine has already recorded at least
   one received byte. A connection that opens, accepts the
   `206 Partial Content` headers, and then streams zero body bytes
   is *never* checked — `lastUpdated` stays at zero forever and
   the watchdog logs once per second but never fires.
2. **Bug B — reset-on-stall reseeds the bug for the retry.** When
   the watchdog *does* fire (i.e. the part received some bytes,
   then went silent for >30s), it resets `part.lastUpdated =
   time.Time{}`. The outer `maxRetries` loop calls `downloadChunk`
   again. The fresh watchdog goroutine starts, reads the
   zero-value `lastUpdated`, and the `IsZero()` guard now suppresses
   the stall check. If the *retry's* connection also opens-then-
   stalls before producing a byte (which is the very pattern
   reported on Cloudflare/R2 byte-range fetches), the goroutine
   blocks indefinitely on a dead TCP stream.

The bug-pair compounds: bug B is what makes bug A user-visible. A
single transient network blip is enough to convert the part from
"watchdog protected" to "watchdog disabled" for the rest of the
session.

## Design — the fix

Three small changes:

1. **Initialize `lastUpdated` at watchdog entry.**
   ```go
   part.lastUpdatedMu.Lock()
   part.lastUpdated = time.Now()
   part.lastUpdatedMu.Unlock()
   ```
   This gives the stall timer a reference point even before the
   first byte arrives. Now a connection that streams zero bytes
   for `stallDuration` correctly trips the watchdog.

2. **Drop the `!lastUpdated.IsZero()` guard.** With the
   initialization above, `lastUpdated` is never zero inside the
   watchdog, so the guard is dead code. Removing it eliminates
   bug A.

3. **Drop the reset-to-zero block on stall.** The watchdog
   returns `errPartStalled` and exits its goroutine; on retry, the
   fresh goroutine will re-initialize `lastUpdated` itself
   (change #1). The reset was actively harmful — it created the
   bug B coupling. Removing it eliminates bug B.

4. **Extract `30 * time.Second` into a package var
   `stallDuration`.** This is a `var`, not a `const`, so tests can
   shorten it. Production default unchanged.

The diff is 12 added / 5 removed lines in `download.go`. Surgical.

## Test coverage

`TestDownloadChunkStallWatchdogFiresWithoutProgress` in the new
`server/download_test.go`:

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Length", "1024")
    w.Header().Set("Content-Range", "bytes 0-1023/1024")
    w.WriteHeader(http.StatusPartialContent)
    w.(http.Flusher).Flush()
    <-r.Context().Done()  // accept connection, write zero body bytes
}))
```

This is **exactly** the failure shape the PR describes: an
`httptest.Server` that returns 206 + flushes headers, then blocks
on `r.Context().Done()` so it never writes a body byte. With
`stallDuration = 200 * time.Millisecond` and a 5s outer context
deadline, the test asserts `errPartStalled` returned and elapsed
time `< 2s`.

The test is well-shaped:

- **Reproduces the actual bug** rather than a proxy for it.
- **Bounds elapsed time on the upper side** so a future regression
  that re-introduces the bug (e.g. someone re-adds the `IsZero()`
  guard) hangs the test up to the 5s context deadline rather than
  reporting a wrong-answer.
- **Restores `stallDuration`** in `t.Cleanup` so the test doesn't
  leak the shortened value to other tests in the same package run.

## Strengths

- **Exemplary PR body.** Names the bug, the symptom, the
  root cause, the mechanism by which the two bugs compound, the
  fix, and the test. Includes a deliberate "behavior-change
  acknowledgment" section noting that the new watchdog *will* now
  fire on cold-CDN initial-byte slowness which the old watchdog
  silently tolerated, with a rationale for why that's the desired
  behavior. This is what a fix-PR write-up should look like.
- **Surgical diff.** Three coupled changes, each tied to one of
  the two bugs and the test-affordance need. No drive-by
  refactoring.
- **Test is the bug.** The reproducer is the failure pattern
  described in the issue, and it would have caught the bug at
  introduction.
- **Race-free fix.** All reads/writes of `part.lastUpdated`
  continue to be under `part.lastUpdatedMu`. No new synchronization
  surface.

## Risks / nits

1. **Stated behavior change is real but probably correct.** The
   PR body explicitly notes:
   > *"With this fix, a 30s no-data window at the start of a part
   > will now surface as `errPartStalled` and the outer retry
   > loop will open a fresh connection."*
   This is the right call. A genuinely slow cold-CDN that takes
   >30s to deliver a first byte is essentially indistinguishable
   from a dead connection from the client's perspective, and the
   outer retry loop treats `errPartStalled` as a soft error
   (`maxRetries=6`), so a slow-but-alive server still converges.
2. **`stallDuration` is now a package var, mutable from any test
   in the package.** Reasonable trade-off, but if multiple tests
   shorten it concurrently (e.g. parallel `t.Run`), they'll
   interfere. A `t.Cleanup` restoration is sufficient for serial
   tests; for parallel tests in the same package, an
   `atomic.Pointer[time.Duration]` or a per-`blobDownload` field
   would be safer. Not a blocker today since the new test is the
   only mutator.
3. **No test for the second bug** (the reset-on-stall reseeding
   bug B). A follow-up test that exercises two consecutive stalls
   on the same `part` and asserts both fire within the bounded
   time would lock that fix in too. Without it, a future refactor
   that re-introduces the reset block can pass the existing test
   while regressing the multi-stall case.
4. **Race-detector note.** PR body mentions pre-existing
   `TestQuantizeModel` and `TestSchedRequests*` race-detector
   failures on unmodified main. Worth confirming with maintainers
   that the new test isn't masking any new race.

## Verdict — merge-as-is

This is one of the cleanest fix-PRs I've reviewed in this drip
series. The bug is correctly identified at the level of *two
compounding flaws*; the fix removes both with minimal mechanical
changes; the test is the actual failure shape. I recommended
`merge-as-is` despite the two follow-up suggestions because they
are genuinely follow-up work (a second test for the multi-stall
case; a parallel-test-safe `stallDuration`), not blockers.

## What I learned

The "watchdog disabled by its own reset path" pattern is a
recognizable shape: any monitor that resets its own state to a
sentinel-suppressed value on detection-event creates a coupling
where the *next* invocation inherits the sentinel and the monitor
is silently disabled. The fix shape — *initialize at entry, never
write a sentinel* — is the canonical form for this class of bug.
The `IsZero()`-guarded watchdog timer is the first cousin of the
"`if !cache.populated { return; } cache.populated = false` after
firing" anti-pattern in subscription/listener code. When reviewing
any timer or watchdog goroutine, the question to ask is: *what is
the state on the very first tick after the goroutine starts, and
does the guard correctly handle that state?* If the answer is "the
guard suppresses the check on the first tick", the watchdog is
useless against cold-start failures, which are exactly the failure
class watchdogs are supposed to catch.
