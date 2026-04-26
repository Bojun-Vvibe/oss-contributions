---
pr: 19445
repo: openai/codex
sha: 8139972dbdc1a48728195fba5fc8531de7bd0d7e
verdict: merge-as-is
date: 2026-04-26
---

# openai/codex #19445 — ci: stop publishing GNU Linux release artifacts

- **Author**: bolinfest
- **Head SHA**: 8139972dbdc1a48728195fba5fc8531de7bd0d7e (MERGED into `main`)
- **Size**: +1/-4 across 1 file (`.github/workflows/rust-release.yml`)

## Scope

Removes the two `*-unknown-linux-gnu` build-matrix entries from the Rust release workflow, leaving only the `*-unknown-linux-musl` Linux targets. Adds one comment on the matrix documenting the intent.

## Specific findings

- **The GNU rows are unused by all release-asset consumers.** PR body confirms "the in-repo release consumers resolve Linux release assets through the MUSL targets." Verified by reading the matrix at `rust-release.yml:69-78` after the diff: only `aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-unknown-linux-musl`, and `aarch64-unknown-linux-musl` remain. So the GNU artifacts have been spending CI minutes producing files that nothing downloads. That's the highest-value category of cleanup: pure dead weight.

- **Comment placement is right.** The new comment at `rust-release.yml:72` lands directly above the surviving `x86_64-unknown-linux-musl` entry: `# Release artifacts intentionally ship MUSL-linked Linux binaries.` That's the line the next contributor will hit when they wonder "where's the gnu target?" — exactly where you want the rationale.

- **Standalone PR shape, well-justified.** The author explicitly argues for keeping this as a *standalone* commit so `git blame` lands on a clear "intentional decision" SHA. That's a good engineering instinct for changes that look like minor cleanups but are actually *policy* — six months from now when someone files "I need a GNU build for some glibc-only dependency", the answer is to revert this commit (or add the row back with a new rationale) rather than spelunking through a multi-purpose PR.

- **MUSL-only is the correct policy for a CLI tool.** MUSL static-links `libc`, which makes the Linux artifacts portable across glibc versions (Alpine, Debian stable, RHEL 7, …). A GNU build is only needed when something in the codebase dynamically links against a glibc-only library — not the case for codex. Reverting the policy later is cheap (re-add 4 matrix rows); paying for unused builds every release is forever.

- **Verification was done by reading the file.** Author's "Verification" section explicitly says "Reviewed `.github/workflows/rust-release.yml` to confirm that the release workflow now only builds Linux release artifacts for `x86_64-unknown-linux-musl` and `aarch64-unknown-linux-musl`." That's the right verification for a workflow-only change — the next release run is the integration test.

- **Already merged.** Post-hoc review: the change is exactly the right shape and size. Nothing I would have asked to be different.

## Risk

Very low. If a downstream consumer was secretly grabbing the GNU asset (unlikely given the author's check), it'll 404 on next release and the consumer will report it. That's a fail-loud failure mode, easily fixed by reverting one commit or pinning to an earlier release. No silent breakage path exists.

## Verdict

**merge-as-is** — already merged. Tiny, intent-documented, single-responsibility infra cleanup that pays for itself in CI time on every release.

## What I learned

"Standalone commit for policy decisions" is a discipline most teams underweight. This PR could have been bundled into any number of release-process refactors and saved the author one round-trip, but they chose the verbose route precisely so the future archaeologist gets a clean signal. The cost is one extra PR; the benefit is forever. Worth copying for any change where the *fact of the change* is more important than the diff itself.
