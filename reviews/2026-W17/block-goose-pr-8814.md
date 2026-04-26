---
pr: 8814
repo: block/goose
sha: 2289bfdcc311732a04e91526d9dad7f100dc2ac4
verdict: merge-as-is
date: 2026-04-26
---

# block/goose #8814 — fix: return error instead of panicking on llama backend init failure

- **Author**: r0x0d
- **Head SHA**: 2289bfdcc311732a04e91526d9dad7f100dc2ac4
- **Size**: ~57 diff lines across `crates/goose/src/providers/local_inference.rs` and `crates/goose-server/src/state.rs`.

## Scope

`InferenceRuntime::get_or_init()` returned `Arc<Self>` and called `panic!("Failed to init llama backend: {}", e)` when `LlamaBackend::init()` failed (missing GPU, unsupported hardware, broken native library). This crashes the entire process with no recovery path — fatal for `goose-server` because `AppState::new()` calls `get_or_init()` unconditionally when the `local-inference` feature is built in, even for users not actively using local inference. PR converts the return type to `Result<Arc<Self>>`, propagates the error via `anyhow::anyhow!`, and updates the two call sites with `?`.

## Specific findings

- `crates/goose/src/providers/local_inference.rs:65` — signature change `pub fn get_or_init() -> Result<Arc<Self>>`. Both internal early-return paths (`return runtime;` → `return Ok(runtime);` at line 67, terminal `runtime` → `Ok(runtime)` at line 91) are correctly wrapped.
- `crates/goose/src/providers/local_inference.rs:83` — the actual fix: `Err(e) => return Err(anyhow::anyhow!("Failed to initialize llama backend: {}", e))` replaces `Err(e) => panic!("Failed to init llama backend: {}", e)`. The error message is improved from "init" to "initialize" — minor wording polish but worth noting it's also a log-grep contract change.
- `crates/goose/src/providers/local_inference.rs:60-65` — the `RUNTIME: StdMutex<Weak<InferenceRuntime>>` static + the in-mutex re-init guard ("the Weak::upgrade() check and LlamaBackend::init() both execute inside this same mutex guard, so there is no window where two threads can both observe `None` and both call `LlamaBackend::init`") is preserved. **Important**: this PR does not change the safety invariant — failure now returns Err *while still holding the mutex*, then drops the mutex on function return; subsequent `get_or_init()` calls will retry the init. Worth confirming retry-on-failure is the intended behavior (vs. caching the failure as a "permanent unavailable" state). The `Weak<...>` will not upgrade after the failure, so the next caller will re-attempt `LlamaBackend::init()`. For a deterministic hardware failure (no GPU) this means every caller pays the failure-init cost. For a transient init failure (e.g. driver hot-restart) this is exactly what you want. Existing comment doesn't address this — could deserve a one-line note.
- `crates/goose/src/providers/local_inference.rs:359` — `LocalInferenceProvider::from_env` already returns `anyhow::Result<Self>`, so the `let runtime = InferenceRuntime::get_or_init()?;` change is a clean `?` propagation. Good.
- `crates/goose-server/src/state.rs:51` — `inference_runtime: InferenceRuntime::get_or_init()?` similarly clean. `AppState::new()` already returned `anyhow::Result<...>`. The fix means a process trying to start `goose-server` without working local-inference hardware now gets an actionable startup error rather than a panic stack trace.
- The `#[cfg(feature = "local-inference")]` guard at line 50 means non-local-inference builds aren't affected — `get_or_init` isn't even called.

## Risk

Very low. The change is a textbook `panic` → `Result` migration. Both callers already returned `Result`, so error propagation is mechanical. Caller-side behavior change: previously the process died with a Rust panic message; now it returns `Err` and lets the binary's top-level error handler render it. Net positive: callers get a chance to log/cleanup/retry.

## Verdict

**merge-as-is** — small, focused, removes a panic that was unjustifiable given both callers already use `Result`. Optional follow-up: a one-line comment about the "every call retries init on failure" semantic of the `Weak`-based caching, since this is a behavior the caller now needs to be aware of (panics didn't have this concern — they killed the process).
