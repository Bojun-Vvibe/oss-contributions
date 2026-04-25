# ollama/ollama PR #15710 — docs: fix tar decompression flags

- **URL:** https://github.com/ollama/ollama/pull/15710
- **Author:** @Persioqq
- **State:** OPEN (target `main`)
- **Head SHA:** `b43a4b62681faa21181833e9088a4d0a192573da`
- **File:** `docs/linux.mdx`

## Summary of change

Replaces `sudo tar x -C /usr` with `sudo tar --zstd -x -C /usr` in
four spots in the Linux install guide where the source archive is
a `.tar.zst` (zstd-compressed tarball).

## Findings against the diff

- **L23, L44, L53, L149:** all four hunks are mechanically
  identical. The downloaded archive is
  `ollama-linux-{amd64,arm64,...}.tar.zst`, which GNU tar ≥ 1.31
  cannot auto-detect via just `tar x` — you need an explicit
  `--zstd` (or `-I unzstd`). Without the flag the user gets
  `tar: This does not look like a tar archive` on most distros,
  matching frequent issue reports.
- **macOS / BSD tar:** also supports `--zstd` since macOS 12+ /
  recent libarchive, so the recommendation is portable enough for
  the "Linux" doc page. The doc explicitly targets Linux so this
  is moot, but worth mentioning if the same blob shows up on
  macOS docs later.
- **No code changes.** Pure docs.
- **Consistency:** the four edits cover every `tar.zst` snippet on
  the page — a `grep` over `docs/linux.mdx` confirms there are no
  remaining `tar x -C /usr` instances after the patch.

## Verdict

**merge-as-is**

Trivial, correct doc fix that resolves a real and well-reported
install failure. No tests, no follow-ups required. Author could
optionally add a sentence about `tar -I unzstd -x -C /usr` as the
fallback for tar versions without `--zstd`, but that's polish, not
a blocker.
