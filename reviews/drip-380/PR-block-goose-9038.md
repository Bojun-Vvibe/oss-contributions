# block/goose PR #9038 — Add Linux Vulkan support for local inference

- URL: https://github.com/aaif-goose/goose/pull/9038
- Head SHA: `89262adab61fbd90f8c67438d53bc586a3291886`
- Size: +104 / -27

## Summary

Wires a `vulkan` cargo feature through `goose-cli` and `goose-server` (delegating to `goose/vulkan` → `llama-cpp-2/vulkan`), enables it on **all** Linux build paths (cross-compile, in-tree cargo, Justfile, Tauri bundle, npm publish, ui/sdk build-native), declares the runtime dep `libvulkan1`/`vulkan-loader` in deb/rpm bundles, and adds a `log_inference_backend_devices()` boot-time probe that lists every non-CPU llama.cpp backend device with memory/free reports. Companion to the existing `cuda` feature; mutually exclusive in spirit (a build is `cuda` *or* `vulkan` *or* CPU-only).

## Specific findings

### Cargo features

- `crates/goose/Cargo.toml:40` — `vulkan = ["local-inference", "llama-cpp-2/vulkan"]`. Mirrors the `cuda` feature one line above. Correct dependency closure.
- `crates/goose-cli/Cargo.toml:77` — `vulkan = ["goose/vulkan", "local-inference"]`. The redundant `local-inference` entry (already implied by `goose/vulkan`) matches the existing pattern for `cuda`. Consistent.
- `crates/goose-server/Cargo.toml:20` — same shape. Consistent.

### Build / CI

- `.github/workflows/build-cli.yml:118-124` — feature-flagging only `*-unknown-linux-gnu` targets:
  ```bash
  FEATURE_ARGS=()
  if [[ "${TARGET}" == *-unknown-linux-gnu ]]; then
    FEATURE_ARGS=(--features vulkan)
  fi
  cross build --release --target ${TARGET} -p goose-cli "${FEATURE_ARGS[@]}"
  ```
  Bash array expansion is correctly quoted; empty-array case (non-Linux targets) expands to nothing. Good.
- `.github/workflows/bundle-desktop-linux.yml:82-85` — adds `libvulkan-dev libvulkan1 glslc` to `apt-get install`. Cross-build correctly receives the headers/loader/shader-compiler trio.
- `.github/workflows/bundle-desktop-linux.yml:116` — switched to unconditional `--features vulkan` (this workflow is Linux-only by file name).
- `.github/workflows/bundle-goose2.yml:444-450, 495` — same package additions and `--features vulkan` on `cargo build --release -p goose-cli --bin goose`. Consistent.
- `Cross.toml:18-19, 35-36` — both `aarch64-unknown-linux-gnu` and `x86_64-unknown-linux-gnu` pre-build install adds `libvulkan-dev:arm64`/`libvulkan-dev` plus `glslc`. **Concern**: `glslc` is added unqualified (no `:arm64` suffix) for the arm64 target — this means the host-arch `glslc` is installed, which is the right call since shader compilation happens at build time on the host, not at link time on the target. Worth confirming with a maintainer the arm64 target actually needs `glslc` rather than `glslc:arm64` (current code is correct for the host-tool interpretation).
- `Justfile:397` — `linux_vulkan_features := if os() == "linux" { "--features vulkan" } else { "" }` correctly gates on host OS for the `bundle-goose2` recipe.
- `ui/scripts/publish.sh:75-81, 116-126` — both inline shell and Dockerfile heredoc gain the three Vulkan packages and the `--features vulkan` flag. Two paths in lockstep — nicely symmetric.
- `ui/sdk/scripts/build-native.ts:60-69` — feature-args gating on `platform.startsWith("linux-")`. Correct conditional for the cross-target script.

### Runtime / packaging

- `ui/desktop/forge.config.ts:95, 111` — adds `depends: ['libvulkan1']` to deb makers and `requires: ['vulkan-loader']` to rpm makers. Right runtime deps for each ecosystem.
- `ui/goose2/src-tauri/tauri.conf.json:53-60` — `linux.deb.depends = ["libvulkan1"]` and `linux.rpm.depends = ["vulkan-loader"]`. **Concern**: Tauri config uses `"depends"` for both deb and rpm — Tauri's RPM config historically uses `"depends"` (which gets translated to `Requires:`) so this is correct, but worth a one-line verify against the Tauri version in use.
- `BUILDING_LINUX.md:11-25` — package install commands updated for Debian/Ubuntu, Arch/Manjaro, Fedora/RHEL/CentOS, openSUSE. **Nit**: Termux section (likely below at line ~30) is not updated — Termux/Android is not a Vulkan target so this is correct, but a one-line note would prevent future confusion.

### Code

- `crates/goose/src/providers/local_inference.rs:88` — adds `log_inference_backend_devices();` after `llama_cpp_2::send_logs_to_tracing(LogOptions::default())` and before constructing the `Self`. Boot-time logging position is right (post-backend-init, pre-runtime-handoff).
- `crates/goose/src/providers/local_inference.rs:160-170, 173-201` — extracts `is_accelerator_device` and `is_non_cpu_device` helpers and rewrites `available_inference_memory_bytes` to use `is_accelerator_device`. Refactor is mechanical, no behavior change.
- `crates/goose/src/providers/local_inference.rs:177-201` — `log_inference_backend_devices` filters non-CPU devices, logs `device_count` if none found (so users with a `vulkan` build but no GPU still get a clear "0 devices" line), and per-device emits `index, backend, name, description, device_type, memory_total_bytes, memory_free_bytes`. Exactly the diagnostic surface a user would want for "is my Vulkan loader picking up my GPU?". Good.
- `ui/goose-binary/README.md:35-39` — explicit doc block calling out build-host vs runtime-host package needs. Helpful.

## Concerns / nits

1. No mention in PR body of mutual exclusivity policy with `cuda` — building both features together would link both backends. May or may not be supported by `llama-cpp-2`. A note in the feature docs would help.
2. The boot-time device log is `tracing::info!` — if the user has tracing filtered to `warn`+, they won't see the diagnostic. Either accept that, or expose a `--list-inference-devices` CLI subcommand for explicit user-driven diagnosis.
3. Termux/Android section in `BUILDING_LINUX.md` left untouched — correct (no Vulkan), but a one-line "N/A on Termux" prevents confusion.
4. `Cross.toml` arm64 `glslc` host-vs-target choice deserves a maintainer confirm.

## Verdict

`merge-after-nits` — clean, well-mirrored across the many build paths, with a useful boot-time diagnostic for end users. Concerns are documentation-grade, not blocking.
