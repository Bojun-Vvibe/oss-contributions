# All-Hands-AI/OpenHands PR #14120 — Removed Architecture diagrams

- **URL:** https://github.com/All-Hands-AI/OpenHands/pull/14120
- **Author:** @tofarr
- **State:** MERGED 2026-04-24 (~17m after open)
- **Base:** `main`
- **Head SHA:** `c848511`
- **Size:** +5 / -348

## Summary of change

Deletes the entire `openhands/architecture/` doc tree:

- `openhands/architecture/README.md`
- `openhands/architecture/system-architecture.md`
- `openhands/architecture/agent-execution.md`
- `openhands/architecture/conversation-startup.md`
- `openhands/architecture/observability.md`

Plus drops `third_party/` from the pre-commit and ruff exclude lists
(those exclusions referenced a directory removed in #14119, the
sibling PR in this stack).

## Findings against the diff

- **`openhands/architecture/agent-execution.md` (deleted)**: contained
  a Mermaid sequence diagram showing the LLM-call flow through
  LiteLLM → llm-proxy.app.all-hands.dev → providers. The diagram is
  the kind of doc that drifts almost immediately after every refactor
  and is rarely consulted at debug time. Removing it is reasonable
  doc-debt cleanup.
- **`dev_config/python/.pre-commit-config.yaml` L5–8 / L37–42**: the
  `^(docs/|modules/|python/|openhands-ui/|third_party/|enterprise/)`
  exclude regex is reduced to drop `third_party/`. This is correct
  bookkeeping after #14119 deleted that directory; leaving the stale
  exclude would not break anything but would mislead the next author
  who greps for `third_party`.
- **`dev_config/python/ruff.toml` L1–2**: same change for ruff. Both
  files are kept in lock-step, which is the right invariant.
- **`dev_config/python/mypy.ini`**: `exclude = (third_party/|enterprise/)`
  becomes `exclude = (enterprise/)`. Three lint configs all updated
  consistently — good.
- **No CHANGELOG / migration note added.** For a docs-only deletion,
  this is acceptable; nothing in user-facing behavior changed. If the
  external website renders these markdown files (e.g., docs.all-hands.dev
  picked them up), a separate cleanup may be needed there, but that's
  a docs-pipeline concern outside this repo.
- **17-minute review cycle** for a -348/+5 diff is reasonable; this is
  a delete-only change against content the author maintains.

## Verdict

**merge-as-is** (already merged)

The change is internally consistent with #14119, the lint configs
are updated in lock-step, and there's nothing to add. The only thing
worth tracking — and it's not a blocker for this PR — is whether
the deleted Mermaid diagrams have a successor anywhere; if the team
intends to publish architecture docs externally, the right home is
probably the docs site, not the source tree.

## What I learned

"Delete the architecture docs" is one of those changes that looks
hostile to newcomers ("how am I supposed to learn the system?")
but is usually correct: stale architecture docs are worse than no
architecture docs because they actively misdirect. The healthier
pattern is to point newcomers at a small number of high-traffic
README/AGENTS.md files plus a `git log --diff-filter=A -- '*.md'`
sweep of recent doc additions, which gives a much more reliable
view of "what does the team currently care about" than a years-old
diagram tree. This PR's velocity (17 minutes to merge) suggests the
maintainers had already paid the cost of letting the docs go stale
and were ready to acknowledge it.
