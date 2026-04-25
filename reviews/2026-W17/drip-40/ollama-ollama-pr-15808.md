# ollama/ollama #15808 — fix: improve error handling for model loading in Scheduler

- **Repo**: ollama/ollama
- **PR**: [#15808](https://github.com/ollama/ollama/pull/15808)
- **Head SHA**: `6f9c1e0a9f4776274da879f2b808d566e0cc0a5f`
- **Author**: famasoon
- **State**: OPEN (+4 / -2)
- **Verdict**: `merge-as-is`

## Context

Tiny but real fix in `server/sched.go`. When the scheduler
detects a model-mismatch after eviction, it `panic`s with a
descriptive error. The pre-PR code panicked **while still
holding `s.loadedMu`**, which meant any subsequent recover /
defer that touched the lock would deadlock the scheduler
goroutine and freeze the process — a misleading symptom for
what is fundamentally an internal-invariant violation.

## Design

Three-line change at `server/sched.go:471-477`:

```go
//  before
if s.activeLoading.ModelPath() != wantPath {
    panic(fmt.Errorf("attempting to load different model after eviction (original %v new %v)", s.activeLoading.ModelPath(), wantPath))
}

//  after
loadedPath := s.activeLoading.ModelPath()
if loadedPath != wantPath {
    s.loadedMu.Unlock()
    panic(fmt.Errorf("attempting to load different model after eviction (original %v new %v)", loadedPath, wantPath))
}
```

Two real changes plus one cosmetic:

1. **Lock release before panic** (`sched.go:473`). The
   `s.loadedMu.Unlock()` immediately before `panic` ensures
   any deferred recover / signal handler / panic-handler
   goroutine that needs the same mutex won't deadlock. In Go,
   `panic` does **not** automatically release locks held by
   the panicking goroutine — they only release if the deferred
   `Unlock()` runs in the unwind path, which means the lock
   must already be deferred (it isn't here, since the function
   does manual `Lock()` / `Unlock()` pairing).

2. **Bind `loadedPath` once** (`sched.go:471`). The original
   called `s.activeLoading.ModelPath()` twice — once in the
   `if` and once in the error message. If `s.activeLoading`
   were swapped between the two calls (it shouldn't be while
   the lock is held, but defence in depth), the error message
   would be misleading. Pinning the value once makes the
   message accurate to whatever caused the comparison to fail.

3. (Cosmetic) Both call sites in the panic message now read
   from the bound variable, keeping the message consistent.

## Risks

- **`Unlock()` before `panic` is correct here, but it's worth
  asking whether `panic` is the right behaviour at all.** A
  scheduler invariant violation is recoverable in principle
  (return an error, log, refuse the load, retry from clean
  state). Panicking takes the whole `ollama serve` process
  down. If the goal is "fail loud during development," the
  PR is fine. If the goal is "production-grade scheduler,"
  this should arguably be a `slog.Error` + `return
  fmt.Errorf(...)` to the caller, so a single corrupt request
  doesn't kill the daemon. But that's a behavioural pivot
  that's out of scope for a bug fix — leave for a follow-up.
- **Order of operations matters.** The `Unlock` happens
  **before** the `panic`'s message string is formatted. That's
  fine in this case because the format args are all on the
  stack (`loadedPath` and `wantPath`, both already-resolved
  strings), but if a future refactor moves the message
  construction inside a function call that re-acquires the
  lock, you've got a re-entrant lock attempt during panic
  unwind. Worth a comment: `// release lock before panic so
  recover handlers can re-acquire`.
- **The fix doesn't address the root question** of how
  `s.activeLoading` ends up with a different model than the
  request expects. If the eviction path's invariant is being
  violated, panic-with-clean-unlock is a band-aid; the
  upstream caller (probably in `Scheduler.processPending` or
  similar) should be tightened so this branch is unreachable.
  Worth filing a follow-up issue.

## Suggestions

- Add a one-line comment above the `Unlock`: `// release
  s.loadedMu before panicking so deferred handlers don't
  deadlock`.
- File a follow-up issue tracking "scheduler model-mismatch
  after eviction is reachable" — the panic itself indicates a
  state-machine bug that should be diagnosed separately from
  the fix to make the panic survivable.
- Consider whether the long-term right answer is to convert
  the panic into a returned error and let the caller decide.

## Verdict reasoning

`merge-as-is`. The patch is small, correct, and strictly
improves crash behaviour without changing the contract. The
panic-vs-error question is a real one but belongs in a
follow-up — keeping this PR scoped is the right call.

## What I learned

Go's `panic` semantics around held locks are easy to forget:
`defer mu.Unlock()` releases on panic, but `mu.Unlock()`
later in the function **does not run** if the function
panics. Mixed manual-and-deferred lock patterns in the same
function are the most common source of this class of
deadlock. The defensive read here ("rebind `loadedPath`
once") is the same idea: don't trust shared state to be the
same between two reads inside a critical section's exit
path. The cleanup also makes the eventual conversion to
`return errors.New(...)` trivial — the lock is already
released, so the caller-error path is one step away.
