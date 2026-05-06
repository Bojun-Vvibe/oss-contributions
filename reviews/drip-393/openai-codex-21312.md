# Review: openai/codex#21312

- **PR:** [release: bundle bwrap with Linux codex DotSlash artifact](https://github.com/openai/codex/pull/21312)
- **Head SHA:** `6259cee9d518e9d190eb30b7ded2a348f7dfeabb`
- **Merged:** 2026-05-06T06:33:14Z
- **Verdict:** `merge-after-nits`

## Summary

Builds a new `codex-<linux-target>-bundle.tar.zst` release artifact that contains both the `codex` executable and `codex-resources/bwrap`, then points the Linux DotSlash outputs at the bundle tarball. Closes the gap where the in-tree `bundled_bwrap.rs` fallback expected a sibling `codex-resources/bwrap` next to the codex binary, but the published DotSlash manifest only materialized the codex executable.

## Specific notes

- `.github/dotslash-config.json:13-19` — Linux x86_64 and aarch64 regexes updated from `^codex-<target>\.zst$` to `^codex-<target>-bundle\.tar\.zst$`. Mac and Windows entries unchanged — correct, those platforms don't ship bwrap.
- `.github/workflows/rust-release.yml:382-390` — bundle assembly is gated on `*linux*` AND `bundle == "primary"`, so the per-PR / non-primary matrix legs don't waste time building it. Files are chmod'd 0755 before tar; permissions survive into the DotSlash extract. Compression is `zstd -T0 -19` matching the existing `.zst` artifacts.
- `.github/workflows/rust-release.yml:416` — extension allow-list extended to include `*.tar.zst` so the pre-existing "convert raw binaries to .tar.gz" step skips the new bundle. Without this guard the bundle would have been double-archived. Good.
- The script copies `bwrap-${{ matrix.target }}` from `$dest/`, which means the bwrap build step must run before this step in the same job. Looking at the surrounding context the existing primary-Linux job already builds bwrap into `$dest`, so this is fine — but a comment to that effect would make the implicit ordering dependency explicit for the next reader.
- The standalone `bwrap` DotSlash output is left in place (called out in the PR description). Backwards-compatible for any consumer that was fetching bwrap directly.

## Rationale

Correct fix at the right layer: DotSlash only materializes the artifact it points at, and the bundled-bwrap fallback in `bundled_bwrap.rs` requires a co-located `codex-resources/bwrap`, so the artifact has to carry both. The matrix gating, permission preservation, and double-archive guard are all handled. Verification listed in the PR (`jq`, Ruby YAML parse) is reasonable for a release-pipeline change.

Nit: drop a one-line comment near the `cp "$dest/bwrap-..."` line noting that the bwrap build step is required to have run earlier in the same job. Not blocking.
