# openai/codex PR #20111 — [codex] Bound advisory system bwrap startup probe

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20111
- Head SHA: `7db3018aa734596f0250e36fef5319e56bd55798`
- State: OPEN, +65/-6 across 2 files

## What it does

Bounds the existing user-namespace probe in `system_bwrap_warning_for_path` with a 500 ms wall-clock timeout so a hung or slow `bwrap --unshare-user --unshare-net --uid 0 --gid 0 --bind / / /bin/true` call never blocks Codex startup. The probe was previously `Command::new(...).output()` (synchronous, unbounded); the diff swaps it for a `spawn()` + `try_wait()` poll loop with a 50 ms poll interval and a 500 ms deadline. On timeout the child is killed, waited, and the probe **returns `true` (no warning)** — preserving the existing fail-open semantic where any unexpected error suppresses the warning rather than emitting a false alarm.

## Specific reads

- `sandboxing/src/bwrap.rs:34-35` — the new constants:
  ```rust
  const SYSTEM_BWRAP_PROBE_TIMEOUT: Duration = Duration::from_millis(500);
  const SYSTEM_BWRAP_PROBE_POLL_INTERVAL: Duration = Duration::from_millis(50);
  ```
  500 ms is generous for a successful `--unshare-user` setup (typically <50 ms on a healthy kernel) and tight enough to not noticeably tax startup. The 50 ms poll interval gives ~10 wakeups per probe — fine. Both are private constants so no public-API surface widening.

- `sandboxing/src/bwrap.rs:88-95` — the timeout branch:
  ```rust
  Ok(None) => {
      if Instant::now() >= deadline {
          let _ = child.kill();
          let _ = child.wait();
          return true;
      }
      thread::sleep(SYSTEM_BWRAP_PROBE_POLL_INTERVAL);
  }
  ```
  Returning `true` on timeout is the right call given the function's contract ("warn iff we know namespaces are broken"). A hang implies "I don't know" → don't warn → fail-open. The two `let _ =` discards on `kill()` and `wait()` correctly tolerate the rare `ESRCH` race where the child exits between `try_wait` returning `None` and the kill firing.

- `sandboxing/src/bwrap.rs:69-77` — the manual `Output` reconstruction:
  ```rust
  let stderr = child.stderr.take().map_or_else(Vec::new, |mut stderr| {
      let mut bytes = Vec::new();
      let _ = stderr.read_to_end(&mut bytes);
      bytes
  });
  let output = Output { status, stdout: Vec::new(), stderr };
  return output.status.success() || !is_user_namespace_failure(&output);
  ```
  Intentionally drops stdout (`Vec::new()`) since `is_user_namespace_failure` only reads stderr. The `read_to_end` here is itself unbounded — if `bwrap` exits but leaves a runaway stderr writer attached to a fork, this read could block past the 500 ms budget. Unlikely in practice (bwrap doesn't fork stderr-writers) but worth a comment that the timeout protects against probe-hang-during-spawn-and-execute, not against stderr-drain-after-exit.

- `sandboxing/src/bwrap_tests.rs:49-65` — the test pins exactly the right invariant:
  ```rust
  #[test]
  fn system_bwrap_probe_times_out_without_reporting_a_warning() {
      let fake_bwrap = write_fake_bwrap("#!/bin/sh\nsleep 1\nexit 0\n");
      // ...
      assert!(system_bwrap_has_user_namespace_access(fake_bwrap_path, Duration::from_millis(10)));
      assert!(started_at.elapsed() < Duration::from_millis(500));
  }
  ```
  Uses a 10 ms timeout against a 1 s sleeper to make the test fast (~50ms worst case). Asserts both correctness (returns `true`, fail-open) and bounded duration (under 500 ms — generous so the test isn't flaky on loaded CI). The 50 ms poll interval is hardcoded into the production code, not parameterized for the test, so on a *very* loaded runner the 500 ms upper bound on the test could be tight — could be relaxed to 1 s to absorb scheduler jitter.

## Verdict: `merge-as-is`

## Rationale

This is a small, well-scoped fix to a real risk: an unbounded synchronous subprocess call on the startup hot path. The replacement is the canonical `spawn` + `try_wait` poll-with-deadline shape, the failure modes (kill failure, wait failure, unexpected `try_wait` error) all converge on fail-open which matches the existing semantic of the function, and the test pins both the correctness and the time-bound. The only nit worth raising — that the post-exit `stderr.read_to_end` is itself technically unbounded — is theoretical and not worth blocking on. Could optionally bump the test's elapsed-time assertion from 500 ms to ~1 s to absorb loaded-CI jitter, but that's tuning, not correctness.
