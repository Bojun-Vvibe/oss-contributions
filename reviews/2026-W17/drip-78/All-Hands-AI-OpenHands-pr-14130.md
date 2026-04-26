# All-Hands-AI/OpenHands PR #14130 — chore(deps): bump pypdf from 6.9.2 to 6.10.2 in /enterprise

- **PR:** https://github.com/All-Hands-AI/OpenHands/pull/14130
- **Head SHA:** `5b56431a8b45f3176320c23a40a97bec4677fa05`
- **Files:** 1 (+12 / -4)
- **Verdict:** `merge-as-is`

## What it does

Pure `enterprise/poetry.lock` bump of `pypdf` from `6.9.2` → `6.10.2`. Single
package, lockfile-only. Per upstream pypdf release notes (linked in the PR
body), `6.10.2` includes a SEC-tagged fix:

> Do not rely on possibly invalid `/Size` for incremental cloning (#3735)

and `6.10.x` introduces limits for FlateDecode parameters and image decoding
to mitigate DoS-via-malicious-PDF. These are exactly the bumps that should
land quickly and unceremoniously — pypdf is the parsing surface for any
user-supplied PDF the agent ingests, and a malformed-PDF DoS or
incremental-cloning corruption is a real, generic, pre-auth-able risk.

## Specific reads

- `enterprise/poetry.lock:11843-11854` — `version = "6.9.2"` → `"6.10.2"`, both wheel and sdist hashes updated. SHA256 values format-clean.
- `enterprise/poetry.lock:12215` — incidental `markers` reordering on `pywin32` (`"sys_platform == \"win32\" or platform_system == \"Windows\""` → `"platform_system == \"Windows\" or sys_platform == \"win32\""`). Logically equivalent, just `poetry lock` reordering. Harmless lockfile noise.
- `enterprise/poetry.lock:14144-14152` — adds 8 new `tree_sitter_c_sharp-0.23.1-cp310-abi3-*` wheel files. These are abi3 wheels for cp310+ across all major platforms. Lockfile is just recording wheel availability that pypi exposed since the previous lock — no actual version bump, no new transitive dep introduction, just `poetry lock` picking up new same-version wheels for additional platforms. Standard.
- No `pyproject.toml` edit needed because the version constraint accepted both 6.9.2 and 6.10.2.

## Why merge-as-is

1. **Single-package security-flavored bump** with upstream-documented SEC fixes. Exactly the dependabot-shaped PR that should land within hours, not days.
2. **Lockfile-only** — zero source code changes, zero test impact, no API surface change.
3. **Patch-version bump within same minor train** (6.9 → 6.10 is technically minor, but pypdf treats `6.x` as a stable train with backwards-compatible additions; their changelog has no `BREAKING:` entries between 6.9.2 and 6.10.2).
4. **Incidental noise** (`pywin32` markers reorder, `tree_sitter_c_sharp` wheel additions) is `poetry lock` housekeeping that would land on any future lockfile touch anyway. Including it here is fine; splitting it is not worth the maintainer round-trip.
5. **Enterprise-scoped**: change is confined to `enterprise/poetry.lock`. Main repo lockfile is untouched, so blast radius is limited to enterprise builds.
6. **No follow-ups needed** beyond the standard "let CI confirm pdf-parsing tests still pass." If there's a `tests/test_pdf_parsing.py` with a couple of representative PDFs, that should already cover it.

Approve and merge.
