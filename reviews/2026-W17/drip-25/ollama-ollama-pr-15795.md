# ollama/ollama PR #15795 — launch: add codex model metadata catalog

- **Repo:** ollama/ollama
- **PR:** [#15795](https://github.com/ollama/ollama/pull/15795)
- **Head SHA:** `0cd8a0a442d2c2d134af083f2caf2a9e3ad20494`
- **Author:** hoyyeva (Eva Ho)
- **Size:** +355 / −15 across 2 files
- **Reviewer:** Bojun (drip-25)

## Summary

`ollama launch codex` flow generates `~/.codex/model.json` for the
selected Ollama model before spawning the Codex CLI, so Codex
picks up real per-model metadata (context window, modality,
reasoning capability) instead of falling back to generic
defaults and printing the "model metadata fallback" warning that
users have been hitting in launched Codex sessions.

The metadata is derived from two sources:
1. The `show` data Ollama already collects for the local model
   (architecture, num_ctx, multimodal flags, etc.).
2. A shared "cloud limits" reference table for known model
   families, so e.g. a local Qwen 3.5 inherits the same
   reasoning / context-window assumptions Ollama's hosted
   variants use.

## Key changes

### `cmd/launch/codex.go` (+123 / −6)

The new logic. Ahead of `exec.Command("codex", ...)`, the
launch flow:
1. Calls into `show` to get the model's reported metadata.
2. Builds a `CodexModelEntry` struct with the context window,
   modalities, and reasoning fields.
3. Writes the resulting catalog to `~/.codex/model.json`.
4. Wires the generated catalog path into the Codex profile
   used by the launch flow so Codex reads it on startup.

The `show`-derived path is the right primary source — it
reflects what the local model *actually* exposes — with the
shared cloud-limits table acting as fallback for fields `show`
doesn't cover (e.g. a model that doesn't advertise its
reasoning capability via metadata).

### `cmd/launch/codex_test.go` (+232 / −9)

Substantial test expansion:
- `TestCodex*` — end-to-end coverage for the catalog
  generation and Codex spawn integration.
- `TestParseNumCtx` — the `num_ctx` parsing helper (handles
  both string and int forms from `show` output).
- `TestModelInfoContextLength` — context-length resolution
  precedence (model-reported vs cloud-limits-fallback).
- `TestBuildCodexModelEntryContextWindow` — the actual
  `CodexModelEntry.ContextWindow` field assembly, exercised
  across model families.

`go test ./cmd/launch -run
TestCodex\|TestParseNumCtx\|TestModelInfoContextLength\|TestBuildCodexModelEntryContextWindow`
is the verification command in the PR body.

## Concerns

1. **Hard-coded path `~/.codex/model.json` is a foreign-tool
   contract.**
   The launch flow writes into another tool's config directory.
   That's the right *operational* shape (Codex reads its
   metadata from a known location), but it's a contract
   couple between Ollama and Codex's expected file layout. If
   Codex changes its metadata-loading path or schema (e.g. to
   `~/.codex/models/*.json` per-model), this code silently
   stops working — the fallback warning the PR is fixing comes
   right back. Worth either:
   - reading the path from a Codex CLI invocation
     (`codex config get models-path` or equivalent) so the
     coupling is explicit and discoverable, or
   - documenting in `cmd/launch/codex.go` the exact Codex
     version range / commit-SHA range that the schema is
     verified against, so the next maintainer knows when the
     contract was last validated.

2. **Overwrite semantics for an existing `~/.codex/model.json`.**
   What happens if the user has hand-edited their
   `~/.codex/model.json` (e.g. to add custom models from
   non-Ollama sources)? If `ollama launch codex` clobbers it,
   that's a silent data-loss footgun. The diff isn't clear
   whether the writer:
   - **replaces** the file entirely (clobbers user edits),
   - **merges** into the existing file (preserves user edits
     for unrelated entries but updates the entry for the
     launched model), or
   - **writes to a sub-path** Codex composes with
     user-managed entries.

   Worth confirming, and worth a backup-on-write semantic
   (`~/.codex/model.json.bak` written before overwrite) if
   the answer is "replaces entirely".

3. **`show`-data parsing is the brittle interface.**
   The `TestParseNumCtx` test handling both string and int
   forms suggests the parser already had to deal with
   unstable JSON shape from `show`. Worth confirming the
   parser is *additively tolerant* — i.e. if `show` adds new
   fields in a future release, the catalog generator
   silently passes them through (or ignores them) rather
   than failing to launch Codex at all. A failing-launch is
   a worse UX than the original "fallback warning".

4. **Cloud-limits reference table needs a lifecycle story.**
   The shared "cloud limits" fallback for known model
   families is a static snapshot. If a model's
   characteristics change upstream (extended context window,
   added modality), the local catalog stays stale until
   someone notices and bumps the table. Worth a follow-up
   issue: source the cloud-limits table from a
   `cmd/launch/cloud_limits.json` so it can be updated
   without a Go-code change, or fetch it from a
   well-known endpoint at launch time with a baked-in
   fallback.

5. **First-launch UX: catalog generation cost.**
   For users with many local models, the launch flow now
   does N `show` calls (one per cataloged model) at every
   `ollama launch codex`. If `show` is cheap, fine. If it
   warms up GPU shaders or paginates over a long manifest,
   first-launch latency could regress noticeably. Worth a
   quick benchmark with 10+ local models to confirm
   launch-time stays reasonable, or caching the catalog with
   an mtime-based invalidation against the local model store.

## Verdict

`merge-after-nits` — solves a real UX paper-cut (the
fallback-warning the PR body calls out is the right symptom
to fix), the `show`-derived metadata is the right primary
source, and the test coverage is substantial (TestCodex,
TestParseNumCtx, TestModelInfoContextLength,
TestBuildCodexModelEntryContextWindow — the four-axis
parametrisation pins the most likely regression vectors).
Three pre-merge asks: clarify overwrite semantics for an
existing user-edited `~/.codex/model.json` (and add a
`.bak` backup if it's full-overwrite), confirm the
`show`-data parser is additively tolerant of new fields,
and add a doc-comment naming the Codex schema version this
catalog targets. Two follow-ups worth filing as issues:
catalog-generation latency benchmark with many local models,
and externalising the cloud-limits reference table.

## What I learned

"Tool A writes into Tool B's config directory to fix a UX
gap in Tool B" is a tempting pattern when both tools ship
together and you control both, but it's a contract debt:
every schema change in Tool B's config format silently
breaks Tool A, and the failure mode (fallback warning is
back) looks like a Tool B regression even though it's a
Tool A integration drift. The defensive shapes are either
(a) make Tool B expose its config path via CLI/RPC so the
coupling is explicit, or (b) pin the contract with a
schema-version field in the written file and a startup-time
check on the writer side that bails if Tool B's expected
schema version has moved. Same lesson applies to anything
that writes into `~/.config/<other-tool>/` or
`~/.<other-tool>/` — treat it like writing to a foreign API,
not like writing to your own state directory. The other
useful pattern this PR demonstrates: "two-source metadata
resolution" (real `show` data primary + shared fallback
table) is a clean way to upgrade gracefully when the
primary source improves over time.
