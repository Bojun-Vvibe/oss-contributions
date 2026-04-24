# ollama/ollama PR #15763 — Prevent system sleep during inference (fixes #4072)

- **Repo:** ollama/ollama
- **PR:** [#15763](https://github.com/ollama/ollama/pull/15763)
- **Head SHA:** `7218f923049a774bdb1a65d8d86a844bd42ed232`
- **Author:** ClawdiaHedgehog
- **Size:** +56 / −0 across 2 files (1 new file, `server/routes.go` patched)
- **Reviewer:** Bojun (drip-26)

## Summary

Adds a refcounted "power lock" around the two HTTP entry points
that drive inference (`GenerateHandler`, `ChatHandler`) so the
host machine does not enter idle sleep while a long generation is
in flight. Implementation is a new `server/power_saver.go` with
two exported functions and a new platform-specific subprocess
strategy:

- macOS → spawn `caffeinate -i -s &` (and reap with
  `pkill -f caffeinate`)
- Linux → spawn `systemd-inhibit --what=idle sleep infinity &`
  (and reap with `pkill -f systemd-inhibit`)

Closes #4072.

## Key changes

### New `server/power_saver.go`

```go
var (
    preventSleepMu sync.Mutex
    preventSleepCnt int
    isActive       bool
)

func AcquirePowerLock() {
    preventSleepMu.Lock()
    defer preventSleepMu.Unlock()
    preventSleepCnt++
    if preventSleepCnt == 1 && !isActive {
        isActive = true
        if runtime.GOOS == "darwin" {
            exec.Command("sh", "-c", "caffeinate -i -s &").Start()
        } else if runtime.GOOS == "linux" {
            exec.Command("sh", "-c", "systemd-inhibit --what=idle sleep infinity &").Start()
        }
    }
}

func ReleasePowerLock() {
    preventSleepMu.Lock()
    defer preventSleepMu.Unlock()
    preventSleepCnt--
    if preventSleepCnt <= 0 {
        preventSleepCnt = 0
        if isActive {
            isActive = false
            exec.Command("pkill", "-f", "caffeinate").Run()
            exec.Command("pkill", "-f", "systemd-inhibit").Run()
        }
    }
}
```

### `server/routes.go:195–198` and `:2106–2107`

```go
func (s *Server) GenerateHandler(c *gin.Context) {
    checkpointStart := time.Now()
    AcquirePowerLock()          // Prevent sleep during inference
    defer ReleasePowerLock()
    var req api.GenerateRequest
    ...
```

(same two lines added to `ChatHandler`).

## What's good

- The motivating use case (long inference jobs being killed by
  display-server idle sleep on a developer laptop) is real and
  affects users daily.
- Platform-specific dispatch via `runtime.GOOS` is the right shape
  for this kind of OS-level inhibitor.
- Refcount + mutex pattern correctly handles concurrent inference
  requests — power stays inhibited until the *last* in-flight
  generation completes.
- Wired in via `defer ReleasePowerLock()` immediately after
  acquire, so panics in handler bodies still release the lock.

## Concerns (substantial — see Verdict)

This patch has multiple serious correctness, security, and
portability issues. I'll enumerate them; collectively they
disqualify the current implementation.

1. **`exec.Command("sh", "-c", "caffeinate -i -s &").Start()` is
   the wrong primitive.** Three sub-issues:

   a. The trailing `&` shells out to background the process *under
      sh*, but `Start()` does not wait, and the parent `sh`
      process exits immediately after backgrounding. The actual
      `caffeinate` PID is now an orphan reparented to PID 1, with
      *no handle in the Go process*. Ollama has no way to send it
      a signal directly — hence the `pkill -f caffeinate`
      kludge in `ReleasePowerLock`.

   b. `pkill -f caffeinate` on macOS will kill **every**
      `caffeinate` process owned by the user, including ones
      started by the user themselves (e.g., a user who runs
      `caffeinate -d` in another terminal during a meeting will
      have their lock killed when ollama finishes inference).
      Same for `systemd-inhibit` — kills any other inhibitors
      the user is running.

   c. The return values of `.Start()` and `.Run()` are discarded.
      If `caffeinate` is not installed (it ships with macOS, but
      it's possible to be missing in stripped containers) or
      `systemd-inhibit` is not present (non-systemd Linux
      distros: Alpine/musl, Void, Devuan, Gentoo with OpenRC),
      the call silently no-ops. The user gets no signal that
      sleep prevention is not actually happening.

2. **Race in `preventSleepCnt == 1 && !isActive`.** This pattern
   exists because `isActive` was added to the structure as a
   second guard, but they're updated under the same mutex — the
   `&& !isActive` is dead unless `Release` decrements the counter
   to zero *without* clearing `isActive`, which it doesn't. The
   `isActive` flag is dead state that confuses the logic. Either
   keep it and use it (e.g., never re-acquire if already active),
   or delete it and gate purely on `preventSleepCnt == 1`.

3. **`preventSleepCnt <= 0` underflow guard.** The patch correctly
   clamps to zero on under-decrement, but never logs why an
   underflow happened. A `Release` without a matching `Acquire`
   means a handler called `Release` twice (e.g., a refactor that
   added a second `defer`). Silent clamping hides the bug. At
   minimum a `slog.Warn(...)` when underflowing.

4. **No proper subprocess management.** The right way to do this
   on macOS is to `exec.Command("caffeinate", "-i")`, capture the
   `*exec.Cmd`, and `cmd.Process.Kill()` (or `Signal(os.Interrupt)`)
   on release. That keeps the PID owned by ollama, does not leak
   to PID 1, does not require `pkill -f`, and does not affect
   other user `caffeinate` processes. Same for `systemd-inhibit`
   (though that one's better invoked via the D-Bus
   `org.freedesktop.login1.Inhibit` API to avoid spawning a
   subprocess at all).

5. **Windows is silently unsupported.** `runtime.GOOS == "windows"`
   takes neither branch — Windows users get no sleep prevention.
   The Windows API for this is `SetThreadExecutionState` with
   `ES_SYSTEM_REQUIRED | ES_CONTINUOUS`, which is a single syscall
   via `golang.org/x/sys/windows`. Not adding it (or at least
   logging that it's unsupported) leaves a documented platform
   silently broken.

6. **No tests.** A refcount + mutex with platform branching is
   exactly the kind of code where a unit test for the counter
   transitions (Acquire → 1, Acquire → 2, Release → 1, Release
   → 0, Release-when-0 → still 0) would catch concern #2 and
   document the invariant. The test does not need to actually
   spawn `caffeinate` — split the spawn behaviour into an
   injectable function.

7. **Possible permission failures, no surfacing.** On a Linux
   system with PolicyKit configured, `systemd-inhibit` may
   require user authorization for `idle` inhibition. The
   `.Start()` call will succeed but the inhibitor will fail to
   register. No way for the user to know.

8. **`Start()` for `sh -c "... &"` leaks file descriptors and a
   short-lived process.** Each `AcquirePowerLock` that flips from
   0 → 1 spawns a transient `sh`. Under sustained traffic where
   the counter oscillates 0↔1 between requests, this churns
   processes. Combined with the absence of `Wait()`, the
   short-lived `sh` becomes a zombie until the next reaper sweep.

9. **`exec.Command(...).Start()` returns an error that is
   discarded** — see 1c. Combined with no logging anywhere in
   this file, debugging "why is my Mac still sleeping during
   ollama inference" is impossible without source-diving.

10. **Style nits in `power_saver.go`:**
    - `import` block uses tab+space inconsistency between
      `"runtime"` and the other imports — `gofmt` will rewrite
      it.
    - Comment on the `var` block reads `// Power management to
      prevent sleep during inference`, which is just a
      restatement of the package-level comment two lines above.
    - `package` doc comment says `// Package server provides the
      Ollama API server` — duplicates the existing doc on
      `routes.go`. Go convention is one package doc, in one
      file.

## Risk

High. The current implementation works *just well enough* to fool a
laptop demo, but in production:

- It silently fails on Windows.
- It silently fails on minimal Linux containers without
  `systemd-inhibit` or on non-systemd distros.
- It interferes with user-owned `caffeinate` / `systemd-inhibit`
  processes via `pkill -f`.
- It leaks orphaned child processes and ignores all subprocess
  errors.

These are not edge cases — they affect ~all Windows users, ~all
Docker users on Alpine-based ollama images, and any developer who
also uses `caffeinate` themselves.

## Verdict

**request-changes**

The intent is right and the placement (`Acquire`/`defer Release`
in the two HTTP handlers) is correct. The implementation needs
substantial rework. Concrete asks:

1. Replace `sh -c "... &"` with `exec.Command(<binary>, <args>...)`,
   capture `*exec.Cmd`, store it in the struct, and signal it
   directly on release. Drop the `pkill` calls entirely.
2. Add a Windows branch using `SetThreadExecutionState` from
   `golang.org/x/sys/windows`.
3. Log (at warn level) when the inhibitor binary is missing, when
   `Start()` errors, and on counter underflow.
4. Delete the dead `isActive` flag (or use it correctly).
5. Add unit tests for the counter transitions, with a swappable
   spawn function so tests don't actually hit `caffeinate`.
6. Document in code that this is best-effort: success of
   `Acquire` does not guarantee the system actually got
   inhibited.

Once those land this becomes a clean merge candidate.

## What I learned

The `pkill -f` reflex is a strong signal that the code lost track
of a PID it should have owned. Whenever the cleanup path can't
target a specific PID, the spawn path is using the wrong primitive.
For "I need to start a long-running helper subprocess and stop it
later", `exec.Cmd` directly (no shell, no `&`, capture `Process`,
signal on cleanup) is the right idiom in Go — `sh -c "... &"`
exists for shell scripts where there's no parent process to hold
the handle, not for Go processes that absolutely have one.

The deeper lesson: OS-level sleep inhibitors are a *capability*
the system gives you for as long as you hold a handle (a process,
a D-Bus inhibitor lease, a Windows execution-state flag). The
moment you release the handle, the capability is gone. So the
in-process state (`preventSleepCnt`) and the OS-level state
(child PID / D-Bus lease / execution flag) must be kept in
sync, which is only possible if you *own the handle*.
