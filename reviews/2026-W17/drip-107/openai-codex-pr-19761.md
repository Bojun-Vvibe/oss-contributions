# openai/codex #19761 — Add app-server DeviceCheck probe

- **Repo**: openai/codex
- **PR**: #19761
- **Author**: jiamingz42
- **Head SHA**: 9103dbca0d8fe41ed5c51e0c2be9c0f1dcaf3a29
- **Base**: main
- **Size**: +227 / −0 across `Cargo.lock` (+1), `app-server/BUILD.bazel`
  (+1), `app-server/Cargo.toml` (+4), new `app-server/build.rs` (+11),
  new `app-server/src/devicecheck_probe.m` (+100), new
  `app-server/src/devicecheck_probe.rs` (+72), `app-server/src/lib.rs`
  (+3), `cli/src/main.rs` (+35).

## What it changes

Adds a hidden, macOS-only `codex app-server device-check-probe` subcommand
for prototyping Apple's DeviceCheck attestation. Three layers:

1. A new Objective-C shim at `app-server/src/devicecheck_probe.m:1-100`
   compiled by `build.rs:1-11` via the `cc` crate (gated on
   `cfg(target_os = "macos")`). The shim dynamically loads
   `DeviceCheck.framework` via `dlopen`/`NSClassFromString` rather than
   linking it, then calls `DCDevice.currentDevice` and
   `generateTokenWithCompletionHandler:` and returns a C struct
   `CodexDeviceCheckProbeResult` containing `supported`, `has_token`,
   `token_length`, and (heap-allocated) `token_base64` /
   `error_description`.
2. A safe Rust wrapper at `app-server/src/devicecheck_probe.rs:7-72`
   that ferries the result into a `DeviceCheckProbeReport` struct
   (`#[serde(rename_all = "camelCase")]`, with `token_base64` marked
   `#[serde(skip_serializing)]` so it never lands in the printed JSON).
   `take_c_string` (`:50-60`) bridges the lifetime — copies the C string
   into a `String` and frees via the exported
   `codex_devicecheck_probe_free`. A non-macOS stub at `:64-72` returns
   `supported=false` with an explanatory error.
3. CLI plumbing at `cli/src/main.rs:471-480` (new
   `AppServerSubcommand::DeviceCheckProbe(DeviceCheckProbeCommand)` with
   a single `--token-out PATH` arg) and the dispatch arm at `:884-902`.
   The token is only written to disk if `--token-out` was supplied; the
   write opens with `OpenOptions::new().write(true).create(true).truncate(true)`
   and on Unix sets `mode(0o600)` (`:893-898`). The pretty-printed
   JSON `report` always goes to stdout. If `report.has_token` is false
   the command exits non-zero via `anyhow::bail!`.

## Strengths

- The 0600 file permission and `serde(skip_serializing)` on
  `token_base64` (`devicecheck_probe.rs:11`) are exactly the right
  defaults for a probe that yields cryptographic material — stdout never
  contains the token, only the explicit `--token-out` path does, and
  even that path is owner-read/write only on Unix.
- Dynamic framework loading at `devicecheck_probe.m:42-49` keeps the
  binary linkable on macOS versions that lack DeviceCheck and on
  non-macOS hosts that produce app-server builds (the `cfg` gate in
  `build.rs:2` and `devicecheck_probe.rs:34/63` mean the Objective-C
  TU is simply absent off-Mac, and the `probe_impl` stub at `:64-72`
  returns a structured report rather than panicking).
- The 30-second timeout via `dispatch_semaphore_wait` at
  `devicecheck_probe.m:84-89` prevents the CLI from hanging
  indefinitely on a misbehaving system extension; on timeout the
  probe returns a structured error rather than blocking.
- The `anyhow::bail!("DeviceCheck probe did not produce a token")`
  at `main.rs:900-901` makes scripted use (CI smoke tests) reliable —
  the JSON is still printed first, so a wrapper can both inspect and
  exit-code-branch.
- The subcommand is `#[clap(hide = true)]` (`main.rs:474`) and routed
  through `reject_remote_mode_for_app_server_subcommand` at `:1517`,
  which is consistent with how other internal probes are gated.

## Risks / nits

- `devicecheck_probe.m:79-83` retains the captured `token` and
  `deviceCheckError` from inside the completion block but does
  `[token retain]` / `[deviceCheckError retain]` — the file is
  compiled without ARC (no `-fobjc-arc` in `build.rs:6-9`), so the
  manual `[token release]` / `[deviceCheckError release]` at `:104-105`
  is correct. But `result.token_base64 = codex_strdup_nsstring(base64)`
  at `:97` runs *before* the release, and the temporary `base64`
  `NSString` is autoreleased by the framework — fine inside the
  `@autoreleasepool` at `:31`, but worth a comment so a future edit
  doesn't accidentally hoist the strdup outside the autorelease pool.
- `objc_msgSend` is cast to typed function pointers at `:51`, `:67`,
  `:80`. On `arm64` (Apple Silicon) `objc_msgSend` is ABI-compatible
  with the typed cast, but on `x86_64` macOS some call sites
  historically required `objc_msgSend_stret` for struct returns. The
  three call sites here all return `id` or `void`, so this is OK
  today, but the file is now an ongoing maintenance hazard if anyone
  adds a struct-returning Obj-C call. A short comment at the top of
  the TU calling out "any new selector with a struct return value
  must use `objc_msgSend_stret` on x86_64" would save future
  debugging.
- The non-macOS stub at `devicecheck_probe.rs:64-72` returns
  `platform: std::env::consts::OS`, but the macOS branch at `:36-44`
  hard-codes `platform: "macos"`. Mostly cosmetic, but consumers
  parsing JSON might want a single source of truth — either both
  branches use `std::env::consts::OS`, or both hard-code their
  string.
- `DeviceCheckProbeReport` has `#[serde(skip_serializing)]` on
  `token_base64` — good — but the field is `pub`. If a third party
  imports this struct via the `pub use` at `lib.rs:103-104` and
  serializes it themselves with their own serializer, the token will
  leak. Worth either making the field `pub(crate)` or wrapping it in
  a `Secret<String>` newtype that has an explicit, documented
  reveal API.
- `build.rs:5-9` doesn't propagate macOS deployment target from the
  Cargo profile, so the compiled `.m` could target a higher minimum
  than the Rust code expects. `cc::Build` honors `MACOSX_DEPLOYMENT_TARGET`
  but doesn't read the workspace's `[package.metadata]` — verify the
  CI macOS images set the env var consistently.
- The `BUILD.bazel` change at `build_script_data = ["src/devicecheck_probe.m"]`
  declares the `.m` as a build-script data dep. That's correct for
  `cargo` (the `rerun-if-changed` covers it), but Bazel sandboxing may
  also need the `.m` exposed via `srcs` of a separate `cc_library` rule
  if the project later wants reproducible Bazel-only builds without
  shelling out to cargo.

## Suggestions

- Make `token_base64` `pub(crate)` (or wrap in a zeroize-on-drop
  newtype) and expose a `consume_token(self) -> Option<String>` method
  so the only way to extract the bytes is through a deliberate API
  call.
- Add a one-line comment in `devicecheck_probe.m` near the
  `objc_msgSend` casts noting the struct-return ABI caveat.
- Consider also writing the `token_length` to stderr on success so
  operators have a quick "did the probe actually generate a token"
  signal without parsing JSON, while keeping the token itself off
  stdout/stderr.

## Verdict

`merge-after-nits` — the probe is well-scoped, the security posture
(0600 file, `skip_serializing`, dynamic framework load, timeout) is
right, and the non-macOS fallback degrades cleanly. The
`token_base64` field visibility and a brief `objc_msgSend` ABI
comment are the only material asks before merge.
