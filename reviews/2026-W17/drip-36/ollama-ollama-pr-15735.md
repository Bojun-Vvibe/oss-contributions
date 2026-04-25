# ollama/ollama #15735 — server: add v2 manifest path

- **Repo**: ollama/ollama
- **PR**: [#15735](https://github.com/ollama/ollama/pull/15735)
- **Head SHA**: `7bcdb250b9ea4cdc104269af5849354c1ead7c79`
- **Author**: pdevine
- **State**: OPEN (+1033 / -197)
- **Verdict**: `needs-discussion`

## Context

Until now, ollama has stored model manifests as plain JSON files
under `~/.ollama/models/manifests/<host>/<ns>/<name>/<tag>` —
named-keyed, *not* content-addressed. Layer/config blobs are content-
addressed (`blobs/sha256-<hex>`), but manifests are not. That mismatch
makes manifest-level dedup, reproducibility checks, and CAS-based
sync impossible. This PR introduces a parallel `manifest-v2/` layout
where manifests are themselves blobs in `blobs/`, and the named-tag
path under `manifest-v2/` is a symlink (or hardlink, or copy) into
`blobs/`.

## Design

Spread across `manifest/manifest.go`, `manifest/paths.go`,
`server/images.go`, and friends. Key pieces:

1. **`writeManifestBlob(data)`** at
   `manifest/manifest.go:264-` (line 264 of the diff). Hashes the
   payload, checks for an existing identical blob with
   `bytes.Equal(existing, data)` short-circuit (a nice no-op for
   re-pulls of the same manifest), and otherwise writes
   atomically via `os.CreateTemp` + rename (the temp lives next to
   the destination so rename is intra-fs). Returns
   `sha256:<hex>` digest.

2. **`LinkManifest(name, digest)`** at lines 215-262 — the
   filesystem-tier dispatcher. Prefers `os.Symlink` with a
   `filepath.Rel`-derived relative target (line 251), falls back to
   `os.Link` (hardlink, line 256), and finally `copyManifestFile`
   (line 260) for filesystems that support neither. The relative-
   symlink choice is correct — it keeps the model dir
   relocatable. Importantly, it `os.Remove`s any existing manifest
   path first (lines 246-248), so a second `LinkManifest` on the
   same name updates the pointer atomically-ish (rm + symlink is not
   actually atomic, but it's no worse than before).

3. **`WriteManifestData(name, data)`** at lines 195-213 wraps
   `writeManifestBlob` + `LinkManifest` and then calls
   `removeLegacyManifestPaths(name)` to clean up the old v1
   location. So every new write migrates that single manifest.

4. **`ParseNamedManifest`** is split into `resolveManifestPath` +
   `parseManifestFile` (lines 115-150). `resolveManifestPath`
   presumably tries v2 first, falls back to v1 — that's the
   downgrade-compat story the PR description mentions. (I'd want to
   read `paths.go` to confirm the resolution order, but the
   docstring at lines 192-194 says "v2 named manifest path …
   prefers symlinks, then hardlinks, then a byte-for-byte copy" so
   the intent is clear.)

5. **In-use accounting** for `RemoveLayers` at lines 75-77 now
   tracks `BlobDigest()` of *other* manifests too, not just their
   layers. This is necessary because a manifest blob can now itself
   be referenced from multiple named-tag symlinks (think `:latest`
   and `:7b` pointing to the same digest), so deleting one tag
   shouldn't GC the manifest blob another tag still uses.

## Risks

- **Rollback hazard**: the PR description says downgrades to older
  versions are supported but any v2-only writes break older
  clients. That's the actual semantic, but it means once a user
  pulls anything on a v2 client they can no longer roll back to
  the previous binary without losing visibility of those models.
  Worth a CHANGELOG entry and probably a one-shot warning on
  first v2 write.
- **Symlink-to-blob durability**: `blobs/sha256-<hex>` is
  reference-counted via `RemoveLayers`'s in-use map. If that walk
  ever misses a manifest (e.g. a partially-written tag during a
  crash), GC could delete a blob that a v2 symlink still points
  at, leaving a dangling tag. The `parseManifestFile` path uses
  `OpenVerifiedManifest` (line 139, 151) which presumably
  re-verifies the digest on read, so the tag would surface as a
  hard failure rather than silent corruption — that's fine, but
  worth confirming the recovery story.
- **Test coverage** — the diff adds substantial test surface in
  `manifest_test.go` (line 603+), `routes_create_test.go` (1311+),
  `routes_delete_test.go` (1518+). I haven't read each, but the
  count is encouraging.
- **`x/imagegen/manifest/`** at lines 1617+ duplicates a chunk of
  this logic into the experimental imagegen tree. Either that
  package should depend on the main `manifest` package, or the
  duplication should be tagged for a follow-up dedupe — copy-paste
  of CAS code is a maintenance hazard.

## Verdict

`needs-discussion` — the design is sound but the rollback story,
the GC-vs-symlink invariant, and the `x/imagegen/` duplication all
deserve explicit answers before this lands. Not request-changes
because nothing is *wrong*; the discussion is about lifecycle and
upgrade contract, not the code itself.
