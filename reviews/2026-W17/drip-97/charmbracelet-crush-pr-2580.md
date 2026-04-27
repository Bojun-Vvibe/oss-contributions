# charmbracelet/crush#2580: feat: comprehensive agent kernel and coordination system

- **Author:** MuX123
- **HEAD SHA:** 1353624f
- **Verdict:** request-changes

## Summary

A 41,781/-1,310 mega-PR claiming to add an "agent kernel" with
context manager, compression, memory, registry, hybrid-brain
coordinator, task scheduler/classifier, permission-grade system,
WebSocket+HTTP server infrastructure, circuit breaker, streaming
monitor, dependency manager, state-machine manager, and a guardian
error-handling system — all in a single PR. The body openly
attributes generation to an AI coding assistant. The diff exceeds
GitHub's 20,000-line diff limit, which alone makes this functionally
unreviewable in the PR UI, and the file list confirms the PR also
ships *committed binary database artifacts* (`crush.db`,
`crush.db-shm`, `crush.db-wal` at the repo root with 0/0 line
counts because they're binary blobs), 17+ design/architecture
markdown files (some in traditional Chinese with English filenames
like `00_構建系統總覽.md`), a 935-line `CHANGELOG.md` net-new file,
multiple top-level vanity markdown files (`MAGICAL.md`,
`CLAUDE_CODE_USAGE_GUIDE.md`, `CLAUDE_CODE_INTEGRATION_TEST_REPORT.md`),
and an entirely new `cmd/hybrid-brain/` go-module with its own
`go.mod`/`go.sum` (740 lines of `go.sum` alone, 1334-line
`main.go`).

This is fundamentally not in mergeable shape regardless of code
quality, and the upstream maintainers (charmbracelet) are extremely
unlikely to land a 41k-line architectural overhaul as a single PR
from a non-core contributor. The request-changes verdict is about
process and shape, not necessarily merit of the underlying ideas.

## Specific feedback

- **Blocker — committed SQLite artifacts:** `crush.db`,
  `crush.db-shm`, `crush.db-wal` at repo root must be removed and
  added to `.gitignore`. WAL files in particular leak ephemeral
  transaction state. These are flagged in the file list with
  `additions: 0, deletions: 0` because they're binary, but they
  *are* in the tree.
- **Blocker — separate-go-module sprawl:** `cmd/hybrid-brain/go.mod`
  + `cmd/hybrid-brain/go.sum` (540 lines) and
  `cmd/claudecode-bridge/go.mod` indicate this PR vendors two
  separate Go modules into the repo. A workspace-style multi-module
  layout is a major architectural decision that needs its own RFC,
  not a side-effect of a feature PR.
- **Blocker — unreviewable size:** GitHub returns
  `HTTP 406: diff exceeded the maximum number of lines (20000)` on
  `gh pr diff 2580`. Split into a series: (1) base-types/interfaces
  PR, (2) message bus, (3) state machine, (4) per-subsystem PRs,
  (5) integration. Each ≤500 net lines.
- **`internal/agent/agent.go` +518/-15** is the highest-risk single
  file — agent.go is the existing core. A 518-line addition needs
  to be its own PR with a clear contract diff against the existing
  agent surface so reviewers can reason about behavior preservation.
- **17 architecture docs** in `docs/architecture/` plus
  `docs/HYBRID_BRAIN_PLAN_v1.0.md`, `docs/IMPLEMENTATION_PLAN.md`,
  `docs/IMPLEMENTATION_PLAN_v1.0.md`, `docs/PROJECT_PLAN.md`,
  `docs/TODO_REVIEW.md`, `docs/STATUS_REPORT_v1.0.md` — these read
  like agent-generated planning artifacts, not maintenance docs.
  Should not land in the canonical repo until the design is
  accepted by maintainers via discussion/RFC.
- **`AGENTS.md` +169/-154** is a near-complete rewrite of the
  project's agent-instruction file — that single edit alone is a
  governance change worth its own PR and discussion.
- **Test additions are large but isolated:**
  `health_check_stress_test.go +560`, `retry_stress_test.go +546`,
  `aggregator_test.go +597`, `dependency_test.go +650`,
  `token_estimator_test.go +441`, `magical_test.go +111`,
  `fork_summarize_test.go +381`, `minimax_integration_test.go +227`
  — large test bodies for code that doesn't exist in upstream;
  recombining tests with the corresponding feature in narrow PRs
  would let each subsystem land independently.
- **`go.sum -285`** at the repo root (the file list shows
  `go.sum: additions 0, deletions 285`) means this PR removes
  existing pinned dependencies from the root `go.sum`. That's a
  red flag — root `go.sum` should typically grow or stay stable
  in a feature PR, not shrink. Likely a regenerate-from-scratch
  artifact that loses provenance.

## Risks / questions

- Provenance: the PR body explicitly says "Generated with Claude
  Code". For a 41k-line architectural change, maintainers should
  ask for a written design rationale from a human author before
  any review effort is spent on the code itself.
- Naming collisions: `cmd/hybrid-brain/` is unrelated to anything
  in the existing crush surface and the name is generic; if the
  concept lands at all it should fit the existing
  `internal/`/`cmd/` taxonomy with a name that maps to a
  user-visible feature.
- Many of the new subsystems (circuit breaker, streaming monitor,
  state machine, dependency manager) overlap with concerns that
  the existing crush codebase already addresses through different
  abstractions. Each new abstraction needs to justify itself
  against the existing one rather than ship in parallel.
- Recommendation: close as `request-changes`, ask the contributor
  to open a discussion thread proposing the agent-kernel direction,
  and only land focused PRs after maintainer alignment.
