# block/goose PR #8814 — fix: return error instead of panicking on llama backend init failure

- Link: https://github.com/block/goose/pull/8814
- Head SHA: `2289bfdcc311732a04e91526d9dad7f100dc2ac4`
- Size: +6 / -6 across 2 files

## Summary

Converts `InferenceRuntime::get_or_init` in `crates/goose/src/providers/local_inference.rs` from `-> Arc<Self>` to `-> Result<Arc<Self>>`, replacing the `panic!("Failed to init llama backend: {}", e)` arm at `:83` with `return Err(anyhow::anyhow!("Failed to initialize llama backend: {}", e))`. Updates the two known callers (`AppState::new` at `goose-server/src/state.rs:51` and `LocalInferenceProvider::from_env` at `local_inference.rs:359`) to propagate the error with `?`.

## Specific-line citations

- `local_inference.rs:65-91`: signature change `pub fn get_or_init() -> Result<Arc<Self>>`. The `Weak::upgrade()` cached-runtime arm at `:67-68` becomes `return Ok(runtime)`, the `Err(e)` arm at `:83` becomes `return Err(anyhow::anyhow!(...))`, and the success arm at `:91` becomes `Ok(runtime)`. The intermediate `LlamaBackend::init()` `Ok(backend)` arm and the `BackendAlreadyInitialized` `expect` are left as-is, which is correct — the second is a real "this can't happen because we hold the mutex" invariant and should still panic if violated, while the *first-time* init failure is a recoverable user-facing error (missing GPU, incompatible llama.cpp version, ENOENT on a model file's transitive dep, etc.).
- `goose-server/src/state.rs:51`: `inference_runtime: InferenceRuntime::get_or_init()?` adds the `?` to propagate up `AppState::new`'s existing `Result`. The `#[cfg(feature = "local-inference")]` gate is preserved, so non-local-inference builds are untouched.
- `local_inference.rs:359`: `let runtime = InferenceRuntime::get_or_init()?` inside `LocalInferenceProvider::from_env`, which already returns `Result<Self>`, so the `?` slots in cleanly.

## Verdict

**merge-as-is**

## Rationale

This is the correct minimal scope for "stop panicking on a recoverable error": the panic was at a hot user-visible boundary (server startup) where a misconfigured local-inference setup would crash the entire Goose server rather than degrade to "local inference unavailable, other providers still work". The new `Err` path lets the caller decide policy. The fact that the `BackendAlreadyInitialized` arm's `unreachable!` is preserved is a good signal — the author distinguished "true invariant violation" from "user-facing init failure" rather than blanket-converting every panic.

The change is mechanical and correct. The only thing I'd note as a follow-up (not blocking) is that `AppState::new`'s caller now sees a different error class for this specific failure (it used to be a process abort, now it's a typed error), so any startup-error logging at the binary entrypoint may want to special-case "local inference unavailable" with a clearer "your local model setup is misconfigured, but the server is still up" message rather than a generic startup-failed log. Not a merge blocker.

