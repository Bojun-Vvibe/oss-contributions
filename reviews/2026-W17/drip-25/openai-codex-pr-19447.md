# openai/codex PR #19447 — ci: release codex-app-server as a standalone binary

- **Repo:** openai/codex
- **PR:** [#19447](https://github.com/openai/codex/pull/19447)
- **Head SHA:** `41bf5e97816863e65ec7a953d81ccfa371e25803`
- **Author:** bolinfest (Michael Bolin)
- **Size:** +173 / −56 across 6 files (CI-only)
- **Reviewer:** Bojun (drip-25)

## Summary

Splits `codex-app-server` out of the existing release `cargo build`
matrix into its own bundle so the IDE-extension / desktop-app
consumers (which don't need the TUI) can pull a smaller, faster-to-
build artifact, and so the new bundle parallelises against the
existing `codex` + `codex-responses-api-proxy` `primary` bundle
instead of being tacked onto its critical path.

Touches three workflow YAMLs, three reusable code-sign actions, and
the DotSlash manifest config — entirely CI plumbing.

## Key changes

### `.github/workflows/rust-release.yml` (+64 / −28)

Becomes "bundle-aware": each macOS and Linux MUSL matrix row now
declares whether it's the `primary` bundle (existing two binaries)
or the new `app-server` bundle (just `codex-app-server`). The
parallelism story is the win — the app-server bundle finishes
materially faster than `primary` because it's a single binary with
a much smaller dependency closure, so freeing it from `primary`'s
job timing should noticeably reduce wall-clock release time.

The risk shape: the matrix expansion has to keep `primary`
behaviour bit-identical. The diff isn't visible inline here, but
the +64/-28 shape is consistent with adding a new `bundle:` key
plus per-bundle `binaries:` list rather than rewriting the existing
job — that's the right shape (additive, preserves prior semantics).

### `.github/workflows/rust-release-windows.yml` (+43 / −19)

Adds the matching `app-server` bundle on Windows, plus the final
packaging job now downloads, signs, stages, and archives
`codex-app-server.exe` alongside the existing release binaries.
The Windows zip-bundling behaviour for the existing `codex` archive
is asserted in the PR body to be unchanged — that's the load-
bearing claim and the one most worth a careful re-read of the
final packaging step. A side-by-side artifact list (pre-PR vs
post-PR) in CI logs would be the easy way to verify.

### Three reusable code-sign actions (+38 / −9 combined)

`linux-code-sign/action.yml`, `macos-code-sign/action.yml`,
`windows-code-sign/action.yml` get a new input that lets each
matrix row declare which binaries to sign. Generalisation is
correct, but this is the spot where I'd most want a default
value asserted: if a row forgets to pass the new input, what
happens? If the action defaults to "sign nothing" silently,
that's a quiet downgrade from the current behaviour where
every row signed `codex` unconditionally. A required input
(no default) is safer here — fail loud, not silent.

### `.github/dotslash-config.json` (+28 / −0)

Adds the `codex-app-server` entry per platform so the DotSlash
manifest publishes it as a first-class downloadable. Pure
addition; reviewer should sanity-check that the SHA256 / URL
template matches the actual artifact name produced by the
Windows packaging job above (a one-character mismatch here
silently 404s for every DotSlash consumer).

## Concerns

1. **No CI dry-run output in the verification section.**
   The PR body says the YAML was parsed locally with `python3` +
   `yaml.safe_load(...)` and the JSON with `json.loads(...)`. That
   verifies syntax, not semantics. The actual smoke test is "did
   the matrix produce the artifacts in the expected places" —
   easiest evidence is a screenshot or paste of the artifact list
   from a release-dry-run on a fork. Worth asking for before merge
   given that release pipelines tend to fail in ways the schema
   parser can't catch (missing secrets, action version drift,
   silent default fall-through).

2. **Code-sign action input default needs to be explicit.**
   If the new "which binaries to sign" input defaults to a
   non-empty list for back-compat, that's reasonable. If it
   defaults to empty, every existing call site that doesn't
   pass the input will start producing unsigned binaries — a
   silent downgrade. A `required: true` on the new input is the
   conservative choice; if back-compat default is wanted,
   document it inline so the next reader doesn't have to infer.

3. **DotSlash manifest URL templating is hand-edited JSON.**
   `.github/dotslash-config.json +28 / −0` is non-trivial.
   Human-edited JSON for a config that's load-bearing for
   downstream consumers wants a schema check in CI (`jq -e` plus
   a small Python validator that resolves every URL template
   against the actual release artifact list). Without that, a
   typo in `codex-app-server` vs `codex_app_server` doesn't
   surface until a downstream user runs `dotslash` and hits a
   404.

4. **Windows DMG path explicitly excluded.**
   PR body says "Kept the macOS DMG focused on the existing
   `primary` bundle; `codex-app-server` ships as the regular
   standalone archives and DotSlash manifest." That's a reasonable
   choice (the DMG is a desktop-app installer, app-server is for
   embedders) but worth a one-line note in the README or release
   docs so embedder consumers know to pull the standalone archive
   instead of looking for it inside the DMG.

## Verdict

`merge-after-nits` — the structural change is correct (decoupling
the app-server bundle from `primary` is a clean parallelism win,
and the code-sign action generalisation is the right factoring).
Two pre-merge asks: a release-dry-run artifact list pasted into
the PR comments to verify the matrix produces what the YAML
claims, and an explicit `required: true` (or documented
non-empty default) on the new "binaries to sign" input. The
DotSlash manifest entry needs a manual cross-check that the URL
template matches the Windows packaging artifact name exactly.

## What I learned

CI matrix splits like this one are deceptively risky because the
failure modes are silent and slow: a broken release pipeline only
shows up when someone tries to cut a release, and a "binary built
but not signed" downgrade only shows up when downstream OS gates
reject the unsigned artifact (which can be days or weeks later for
embedders who pin to a specific release). The right defensive
posture is to make the new inputs `required: true` rather than
adding a permissive default, and to gate the merge on a
release-dry-run from a fork. The same shape applies to any change
that touches a reusable action's input surface: think of it like
a public API change, not a CI-internal refactor.
