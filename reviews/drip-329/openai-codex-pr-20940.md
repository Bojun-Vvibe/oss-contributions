# openai/codex #20940 — [codex] Split app-server request processors

- SHA: `41258575c60dc98ab268f2aba9ae4e0e3f3c193d`
- State: OPEN, +13,781/-12,664 across ~40 files
- Key files: `codex-rs/app-server/src/codex_message_processor.rs` (deleted, -10,776), `codex-rs/app-server/src/message_processor.rs` (+571/-583), `codex-rs/app-server/src/request_processors.rs` (new, +481), `codex-rs/app-server/src/request_processors/{thread_processor,turn_processor,thread_lifecycle,account_processor,...}.rs` (new)
- Validation in PR body: `cargo test -p codex-app-server`, `cargo check -p codex-app-server`, `just fix -p codex-app-server`

## Summary

Mechanical refactor: shred the ~10.7k-line `CodexMessageProcessor` god-object into command-prefix-scoped request processors under `app-server/src/request_processors/`. Removes `config_api.rs`, `device_key_api.rs`, `external_agent_config_api.rs`, `fs_api.rs` API-wrapper layers by inlining their handlers into the new processors. Shared lifecycle / summary / token-replay / error-mapping kept where ≥2 processors use them; single-use helpers inlined.

## Notes

- `codex-rs/app-server/src/codex_message_processor.rs` (-10,776 LOC) — full deletion. No leftover re-exports; new owners are imported per call site in `message_processor.rs:12077-12095`. Mechanical split confirmed.
- `codex-rs/app-server/src/message_processor.rs:12053-12172` — top of file rewritten: removes `CodexMessageProcessor`, `ConfigApi`, `DeviceKeyApi`, `ExternalAgentConfigApi`, `FsApi` field types from the struct; replaces with the new per-prefix processors. Also drops the chunk of `codex_app_server_protocol::*` imports that now live inside their owning processors (`AppListUpdatedNotification`, `ExternalAgentConfigImport*`, `InitializeResponse`, `ModelProviderCapabilitiesReadResponse`, `ServerNotification`, etc.). Symmetric.
- `codex-rs/app-server/src/message_processor.rs:12162-12172` — `MessageProcessor` struct contraction is the riskiest readable area: from monolithic `codex_message_processor: CodexMessageProcessor` field down to the prefix-scoped processors. Reviewers should focus here: any field that used to be `pub(crate)`-accessible from `codex_message_processor` modules now needs the new processor's surface to expose the same accessors. The PR claims `cargo check -p codex-app-server` is green, which would catch most silent breakages, but a runtime-only path (e.g., a notification fan-out that used to share a `Mutex` reference) might slip through compile checks.
- `codex-rs/app-server/src/request_processors/thread_processor.rs` (+3,999) — single largest new file. By itself this is still a god-object; the split is "break out by command prefix" not "break out by concern". Reasonable as a first cut, but the title's "Split" understates how concentrated thread/* commands remain. Worth tracking a follow-up to break thread_processor by intent (lifecycle vs. turn vs. summary already partially extracted into siblings).
- `codex-rs/app-server/src/request_processors/turn_processor.rs` (+1,126) and `thread_lifecycle.rs` (+762) — the fact that turn handling is its own file but still imports lifecycle helpers from a sibling shared module is the right call; preserves the "shared only when ≥2 callers" rule the PR body states.
- `codex-rs/app-server/src/request_processors/{config,device_key,external_agent_config,fs}_processor.rs` — these absorb the deleted `*_api.rs` wrapper files. The diff for `device_key_processor.rs` is +120/-45 (vs new file +120/-0), meaning some pre-existing content was edited rather than purely added — confirms the wrapper logic was lifted in, not just renamed.
- `codex-rs/app-server/src/request_processors/{command_exec,external_agent_config,thread,thread_summary}_processor_tests.rs` and `request_processors/thread_processor_tests.rs` (+1,146) — tests co-located with their processors via `_tests` files (matching the PR's "moved processor tests to `_tests` files" claim). Fits Rust's flat tests-as-modules convention used elsewhere in codex-rs.
- `codex-rs/app-server/src/request_processors.rs:1-481` — new top-level dispatcher / re-export hub. Worth confirming that ordering of `pub mod` declarations matches the prefix groups in `message_processor.rs` to keep grep-ability.
- `codex-rs/app-server/src/lib.rs:1` — only -5/+1; the deleted `pub mod codex_message_processor;` is replaced by `pub mod request_processors;`. Clean module-level swap.
- Risk: a refactor at this scale (+13.7k/-12.6k) is hard to review by line; the burden falls on `cargo test -p codex-app-server` covering enough of the surface. PR body lists only that single command. Recommend confirming the suite includes tracing-test (renamed `message_processor/tracing_tests.rs` → `message_processor_tracing_tests.rs`) and that no test was inadvertently dropped from the Cargo manifest.
- No new public protocol surface — this is internal-only. Lower blast radius than the LOC count suggests.

## Verdict

`needs-discussion` — mechanically sound and the compile-and-test gate is honored, but a +13.7k/-12.6k refactor with one validation line in the description deserves: (a) explicit confirmation of test count before/after, (b) a follow-up issue scoped at sub-splitting `thread_processor.rs` (3,999 LOC), and (c) reviewer sign-off from someone fluent in the deleted notification fan-out paths. After those, this is `merge-as-is`.
