# BerriAI/litellm PR #26602 — `fix(docker)`: exclude `docker/build_from_pip/` from runtime image

- **PR**: https://github.com/BerriAI/litellm/pull/26602
- **Author**: @ishan-modi
- **Head SHA**: `a653429b`
- **Size**: +1 / −0
- **Files**: `.dockerignore`

## Summary

Adds one line to `.dockerignore` so the `docker/build_from_pip/` directory — a CI smoke-test fixture that contains a sample `requirements.txt` pinning a deliberately-old litellm version for "build from PyPI" reproducibility — stops shipping inside the published runtime image. The fixture is never imported, executed, or read at runtime; the only effect of it being present in the image was that downstream CVE policy gates would scan the embedded pin, see the deliberately-old version, and flag the image as vulnerable even though the actual installed package is patched.

## Verdict: `merge-as-is`

Single-line `.dockerignore` addition with a clear, well-scoped rationale. The directory's purpose is fixture-only, the failure mode is real (false-positive CVE flag), and the fix is the minimum-surface change.

## Specific references

- `.dockerignore:11` — slots the new pattern between the existing `docker/Dockerfile.*` exclusion and the `# Claude Flow generated files` block:
  ```
  docker/Dockerfile.*
  + docker/build_from_pip/
  ```
  Trailing slash means "directory only" — won't accidentally match `docker/build_from_pip` if it existed as a file. Pattern is anchored relative to build context root, which is correct because that's where the directory lives.
- The existing `tests` line two rows up shows the precedent: anything that's CI-scaffolding-only should not be in the runtime image.

## Notes

1. The PR description names "CVE policy gates on downstream builds" as the failure mode but doesn't link a specific scanner output. If there's a public DSO/Trivy/Grype trace, dropping it in the PR makes the rationale undeniable for future archaeologists asking "why is this in `.dockerignore`?"
2. Worth a one-line comment in `.dockerignore` (e.g. `# CI-only fixture; embedded pin trips downstream CVE scans`) — `.dockerignore` files have no comment convention difficulty (they accept `#` comments) and this is exactly the kind of "why" that gets lost in two months.
3. Consider auditing the rest of `docker/` for anything else that's CI-only and not runtime-used (e.g. test compose files, sample configs). Not blocking — just "while you're in there".

## What I learned

`.dockerignore` is consistently underused as a security/compliance surface — most projects treat it as "speed up build" only, but excluding fixture artifacts that contain deliberately-old version pins is the right defense against scanners that recursively crawl image filesystems looking for `requirements.txt` / `package.json` / `Cargo.lock` shapes regardless of whether they're "active". The general rule: if a file in your repo would embarrass you when a CVE scanner finds it inside your published image, and nothing at runtime needs it, `.dockerignore` it.
