---
pr_number: 15755
repo: ollama/ollama
head_sha: b7da254f46a569f32ef228844c51b1f65f024165
verdict: merge-after-nits
date: 2026-04-25
---

# ollama/ollama#15755 — metal: harden against ggml init failures + GPU-discovery serialization

**What changed.** 11 files / +450 / −107. Three intertwined fixes:

(1) **Metal tensor-API retry.** `llm/server.go` adds `ShouldRetryWithMetalTensorDisabled(err, status *StatusWriter) bool` (line 442). `discover/runner.go` introduces `bootstrapDevicesWithMetalRetry(firstAttemptCtx, retryParentCtx, timeout, ...)` (line 429) which, on first-attempt failure matching the heuristic, retries with `GGML_METAL_TENSOR_DISABLE=1` injected. Successful Metal devices have `RunnerEnvOverrides["GGML_METAL_TENSOR_DISABLE"] = "1"` recorded so subsequent model loads inherit the workaround (`recordPersistentRunnerEnv`, line 470).

(2) **`StatusWriter` race fix.** `llm/server.go` line 388 — replaces the prior pattern of separate `cmd.StdoutPipe()` + `cmd.StderrPipe()` goroutines (which copied the same `out io.Writer` from two routines without synchronization) with `cmd.Stdout = out; cmd.Stderr = out` directly, with a comment that "os/exec serializes Write calls when shared". The `s.status.LastErrMsg` field is replaced with `LastError()` / `SetLastError(...)` accessor methods — implies the underlying field is now mutex-guarded.

(3) **GPU-discovery dummy-load synchronization.** Body explains: previously, concurrent `/info` calls would each trigger a dummy load against the runner concurrently, causing a 30s hang. Fix synchronizes the dummy-load inside the runner so concurrent callers queue and reuse the result.

Also: `defer cancel()` was moved out of the loop iteration in `GPUDevices` (line 113) — the previous code captured `cancel` in a `defer` inside a loop, which leaks contexts until function return.

**Why it matters.** Fixes #15734. Real symptom: hard runtime crash on systems where the Metal probe says "tensor API supported" but actual kernel coverage is incomplete. Class of bug typical of probe-vs-runtime drift in vendor frameworks.

**Concerns.**
1. **`defer cancel()` removal at line 113.** Now uses explicit `cancel()` after `bootstrapDevicesWithMetalRetry`. Correct and necessary (the previous `defer cancel()` in a for-loop is a known Go pitfall). But the second-pass branch at line 151 doesn't show a defer/explicit cancel pair in the truncated diff — confirm the second-pass `ctx2ndPass` is also explicitly cancelled at the right scope.
2. **`runDiscovery` closure** (line 433) captures `ollamaLibDirs` from the enclosing scope and accepts `extraEnvs` as a parameter. The retry path constructs `retryEnvs` by copying then mutating — fine, but `recordPersistentRunnerEnv` is called twice on success-after-retry: once with original `extraEnvs` (no-op because `GGML_METAL_TENSOR_DISABLE != 1`) and once with `retryEnvs`. Slightly wasteful guard, fine in practice.
3. **`StatusWriter.LastError()` / `SetLastError(...)`** — the diff replaces direct field access with methods, but I can't see the implementation in the 250-line slice. If `LastError()` is just `return s.LastErrMsg` without a mutex, the fix is incomplete. The comment "os/exec serializes Write calls when shared" only protects against interleaved bytes; the `LastErrMsg` mutation in the `Write` parser must still be atomic vs reads from `NewLlamaServer`. Verify.
4. **`ShouldRetryWithMetalTensorDisabled`** — the name implies a stable contract but the matching heuristic (looking at `status.LastError()` substring) is brittle to ggml log-line wording changes upstream. If ggml ever rephrases the error string, retry silently stops engaging. Pin the matched substrings in a dedicated constant + test against captured fixture log lines.
5. **`runtime.GOOS != "darwin"` early-return** is correct but `GGML_METAL_TENSOR_DISABLE` is a build-time/runtime env Metal-only knob anyway. Belt-and-suspenders.
6. **Test for `recordPersistentRunnerEnv`** (`runner_test.go` line 113) covers Metal vs CUDA selectivity — good. No test for the retry-engagement decision itself; add one fixture-driven test that feeds a captured failure-mode log into `ShouldRetryWithMetalTensorDisabled`.

Real crash fix tackled holistically. Land after substring-fixture test (#4) and the `LastError()` mutex audit (#3).
