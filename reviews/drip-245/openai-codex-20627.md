# PR #20627 — fix: cargo deny

- Repo: openai/codex
- Head: `c3a09c54a7f0b260cbcbd1bf1aa9c763bfb5eb82`
- URL: https://github.com/openai/codex/pull/20627
- Verdict: **merge-as-is**

## What lands

Two-part fix to keep the Rust supply-chain CI green:

1. Pin `dtolnay/rust-toolchain` to `1.93.0` in `cargo-deny.yml`,
   `rust-release.yml`, and `cargo-audit.yml`. Previously two of the
   workflows tracked `stable` (which had floated past 1.93.0 and
   triggered a `deny`-level warning chain) and one was pinned to `1.92`.
2. Add ignore entries for `RUSTSEC-2026-0118` and `RUSTSEC-2026-0119`
   (hickory-proto via `rama-dns`/`rama-tcp`) to both `audit.toml` and
   `deny.toml` until rama updates to hickory 0.26.1 or hickory-net.

## Specific findings

- `.github/workflows/cargo-deny.yml:20,25` swaps both the toolchain
  install action commit and the `rust-version` input from `stable` to
  `1.93.0`. This is the right pair — leaving `rust-version: stable` in
  the action input while pinning the install action would re-introduce
  drift on the runner side.
- `.github/workflows/rust-release.yml:23` bumps from `1.92` (commit
  `c2b55edf...`) to `1.93.0` (commit `a0b273b4...`). This is the same
  toolchain commit referenced by the other two workflows — important
  because mismatched toolchains across release vs. CI are a recurring
  source of "passes CI, fails release-build" issues.
- `.github/workflows/cargo-audit.yml:20` had been `@stable` (a floating
  branch ref, not even a commit pin). Switching to the pinned commit
  `a0b273b4...` # 1.93.0 closes a small but real supply-chain hole: a
  compromised `dtolnay/rust-toolchain@stable` ref would have been
  consumed silently.
- `codex-rs/.cargo/audit.toml:9-10` and `codex-rs/deny.toml:81-82` add
  the two RUSTSEC-2026 entries with `reason` strings that explicitly
  call out `codex-network-proxy` as the consumer and "DNSSEC features
  are not enabled" as the mitigation argument for `RUSTSEC-2026-0118`.
  That's the exact kind of context future maintainers need to decide
  whether the ignore is still safe.

## Verdict rationale: merge-as-is

- Each ignore entry has a tracked-removal condition ("remove when rama
  updates to hickory 0.26.1 or hickory-net") so this isn't a
  set-and-forget escape hatch.
- The toolchain pinning fixes a real CI noise source AND tightens
  supply-chain posture (no more floating `@stable` ref). Both changes
  in one PR is fine because they're all in `.github/workflows/` and
  `.cargo/` config files — no production code is touched.
- The DNSSEC mitigation argument for the hickory CVEs is credible:
  `codex-network-proxy` doesn't enable DNSSEC features per the comment,
  so the affected code paths are unreachable in this build.
- Worth a follow-up issue (not blocking) to track the rama→hickory
  upgrade so the ignore entries can be removed; the comment hints at it
  but a real issue would be more durable.
