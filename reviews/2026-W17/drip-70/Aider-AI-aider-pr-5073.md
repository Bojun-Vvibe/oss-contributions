# Aider-AI/aider#5073 — feat: add SAST scanning workflow with Semgrep and OSV-Scanner

- **Repo**: Aider-AI/aider
- **PR**: [#5073](https://github.com/Aider-AI/aider/pull/5073)
- **Head SHA**: `71b812e4f599`
- **Author**: GeoDerp
- **Base**: `main`
- **Size**: +68 / −0, 1 file (`.github/workflows/sast-scan.yml`)

## Context

Adds a new GitHub Actions workflow that runs Semgrep (with five rule packs:
`p/python`, `p/owasp-top-ten`, `p/cwe-top-25`, `p/security-audit`,
`p/secrets`) and OSV-Scanner against the repo on every PR to `main`, every
push to `main`, and weekly at Sunday 00:00. Both jobs upload SARIF to the
GitHub code-scanning surface.

## Change

Single new file `.github/workflows/sast-scan.yml` with two jobs:

1. **`semgrep`** — runs in container `semgrep/semgrep:1.66.0` on
   `ubuntu-24.04`, calls `semgrep scan --config=p/python
   --config=p/owasp-top-ten --config=p/cwe-top-25 --config=p/security-audit
   --config=p/secrets --sarif --sarif-output=semgrep.sarif .` with
   `continue-on-error: true`, then uploads via
   `github/codeql-action/upload-sarif@v3.25.0` under category `semgrep`.
2. **`osv-scanner`** — runs `google/osv-scanner-action@v2.3.5` with
   `--min-severity=7.0 --recursive ./`, also `continue-on-error: true`,
   uploads under category `osv-scanner` with `if: always()` so failed
   scans still publish partial SARIF.

Both jobs request `security-events: write` and `actions: read`
permissions; the top-level `permissions:` block correctly drops the
default `contents: write`.

## Strengths

- **Minimal-permission workflow** — explicit top-level `contents: read,
  security-events: write, actions: read` is the right shape for a
  scan-only job and follows GH security hardening guidance.
- **Pinned action versions** (`actions/checkout@v4.1.2`,
  `github/codeql-action/upload-sarif@v3.25.0`,
  `google/osv-scanner-action@v2.3.5`, container
  `semgrep/semgrep:1.66.0`) — full SemVer pins, no floating tags.
  Better than the typical `@v4` pattern.
- **`continue-on-error: true`** on the scanner steps means the scan
  result publishes to code-scanning even when findings exist, instead
  of failing the PR check on the first violation. That's the right
  default for a *new* workflow — start in observe mode, ratchet to
  blocking once the baseline is clean.
- **Two-category SARIF upload**
  (`category: semgrep` / `category: osv-scanner`) keeps the
  code-scanning UI's per-tool deduping correct.

## Risks / nits

1. **Trailing whitespace + missing newline at EOF**: the diff shows
   `continue-on-error: true ` (trailing space) on lines 35 and 56, and
   `\ No newline at end of file` on the last line. Cosmetic but trips
   most YAML linters. Easy fix.
2. **`min-severity=7.0` for OSV-Scanner is aggressive**: it filters out
   medium-severity vulns (CVSS 4–6.9). For a Python project with a deep
   transitive dep tree, that's reasonable as a first cut, but worth a
   comment in the workflow file justifying the threshold (or making it
   a workflow input). Without a justification, the next reviewer will
   want to lower it.
3. **Five Semgrep rule packs is a lot**: `p/python` + `p/owasp-top-ten` +
   `p/cwe-top-25` + `p/security-audit` overlap heavily (the first three
   are mostly subsets of `security-audit`). Run once with all five, look
   at the SARIF, and prune to two or three to cut noise. `p/secrets` is
   the only one that's clearly orthogonal and worth keeping standalone.
4. **No baseline acknowledgment**: a fresh SAST workflow on a
   long-lived Python codebase will surface dozens of findings on the
   first run. Without a `--baseline-ref` or a documented "we accept
   the existing baseline" note, the code-scanning tab will flood. The
   `pull_request` trigger only scans the PR diff in newer Semgrep
   versions if you pass `--baseline-ref=$GITHUB_BASE_REF` — without
   that flag the PR run rescans the full tree. Either add the flag
   or set `pull_request` to `paths-ignore` the noisy directories.
5. **Schedule is `0 0 * * 0`** — Sunday 00:00 UTC. Fine, but cron
   minute 0 collides with millions of repos. Adding a small jitter
   (`0 3 * * 0`) is a kindness to GH's job queue.
6. **No `concurrency:` group**: rapid PR pushes will queue multiple
   Semgrep container starts. Adding
   `concurrency: { group: sast-${{ github.ref }}, cancel-in-progress:
   true }` saves CI minutes.

## Verdict

**merge-after-nits** — the workflow is structurally correct and
minimum-privilege, but should be tweaked before merge to add a Semgrep
baseline-ref (or paths-ignore) so the first PR run after merge doesn't
flood the code-scanning tab, plus the trailing-whitespace / EOF fixes
and ideally the `concurrency:` group. The rule-pack pruning can wait
for a follow-up after the first scan reveals the actual signal-to-noise.

## What I learned

`continue-on-error: true` on the *scan* step (not the upload step) is
the right pattern for "start in observe mode" SAST onboarding —
findings flow to the code-scanning UI without breaking the merge gate,
and you flip the flag off later. The alternative (running the scanner
in PR comments only) defers the SARIF history, which makes baseline
ratcheting much harder six months in.
