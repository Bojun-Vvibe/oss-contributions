# ollama/ollama#15726 — fix: resolve gateway launch timeout on Windows
**SHA**: `6abdd3ec` · **Author**: UniquePratham · **Size**: +3/-3 across 2 files

## What
Two-line change replacing `localhost:%d` with `127.0.0.1:%d` in the
gateway launch path: once for the readiness-probe address, once for
the user-facing URL printed on launch. The motivation is well-known
on Windows — `localhost` resolution can stall for seconds when the
default name-resolution order tries IPv6 first and the gateway is
only listening on the IPv4 loopback (or vice versa). Forcing the
literal v4 loopback bypasses the resolver entirely.

## Specific cite
- `cmd/launch/openclaw.go:168` —
  `addr := fmt.Sprintf("127.0.0.1:%d", port)` (was `localhost:%d`).
  This is the actual fix: this string is fed to the readiness probe
  shortly after, and a 30-second name-resolution stall here was the
  reported symptom.
- `cmd/launch/openclaw.go:344` — the printed URL also flips to
  `http://127.0.0.1:%d`. Cosmetic but consistent; matches what the
  probe is actually reaching.
- `cmd/launch/openclaw_test.go:2138` updates the assertion from
  `"localhost:9999"` to `"127.0.0.1:9999"` — confirms the test was
  pinning the printed string and would have caught a silent revert.

## Verdict: merge-as-is
Smallest-possible fix for a well-understood class of Windows
resolver bugs. The test was already covering the printed-URL string,
so the fix is regression-protected. No need to over-engineer this
into a config option — `127.0.0.1` is what the gateway binds to in
practice and forcing the literal eliminates the name-resolution
variable entirely.
