# Review — BerriAI/litellm#26989

- PR: https://github.com/BerriAI/litellm/pull/26989
- Title: feat(newrelic): Add New Relic extension
- Head SHA: `1962b2d32f943b20bce8df803f4007696a7ff54d`
- Size: +2304 / −1 across 16 files
- Verdict: **request-changes**

## Summary

Adds a parallel "New Relic" deployment flavor under `docker/newrelic/`:
a derived Docker image based on `docker.litellm.ai/berriai/litellm`
that wraps the litellm entrypoint with `newrelic-admin run-program`,
plus a duplicated supervisord config and entrypoint script for the
`SEPARATE_HEALTH_APP=1` mode.

## Evidence

- `docker/newrelic/Dockerfile:1-18` — derives from the published
  litellm image, runs `python -m ensurepip && python -m pip install
  --no-cache-dir 'newrelic>=12.1.0,<13'` as root, then overrides
  `ENTRYPOINT` to `/app/docker/newrelic/entrypoint.sh`. The base-tag
  is parameterized via `BASE_TAG=main-stable`.
- `docker/newrelic/entrypoint.sh:1-10` — shell script with a
  comment `⚠️ KEEP IN SYNC: Changes to docker/prod_entrypoint.sh
  logic should be reviewed here`. This is an explicit drift hazard.
- `docker/newrelic/supervisord.conf:1-50` — copy of the existing
  `docker/supervisord.conf`, with the same `KEEP IN SYNC` comment.
  "This config is identical to supervisord.conf except the main
  program command always wraps with newrelic-admin (no
  conditionals)."
- The remaining ~2200 LOC across 16 files presumably contain the
  README, CI build entry, and possibly sample configs (not fully
  inspected line-by-line for this review).

## Why request-changes

1. **Two `KEEP IN SYNC` annotations are not a synchronization
   strategy.** The PR ships with a guarantee that the duplicated
   shell + supervisord files will drift the moment anyone touches
   `docker/prod_entrypoint.sh` or `docker/supervisord.conf` without
   knowing about the New Relic mirror. At minimum this needs:
   - a CI check that diff-asserts equivalence (e.g. a script that
     generates the New Relic variant from the canonical files at build
     time, then checks `git diff --exit-code`), or
   - inverting the relationship so the canonical entrypoint sources a
     hook (`/app/docker/hooks/wrap.sh`) and New Relic supplies its
     wrapper there, eliminating the duplicate.

2. **`USER root` then unconditional `python -m ensurepip` runs at
   image build time without any `USER` revert at the end.** If the
   base image dropped privileges, this image now runs as root.
   Please end the Dockerfile with the same `USER` directive the base
   image used (or document why root is required).

3. **`newrelic>=12.1.0,<13` is unconstrained on patch.** Reproducible
   builds want a `==` or at least a lockfile entry. For an
   observability agent that runs in-process this matters: a bad
   patch release of the agent can pin-prick produce hangs at
   shutdown.

4. **No tests / smoke check.** A 2.3kLOC PR adding a deployment
   surface should at minimum have a "image builds and `--help`
   exits 0" job in CI.

## What I'd want to see before re-review

- A single source of truth for the entrypoint and supervisord
  configs, with the New Relic flavor expressed as a delta (overlay
  file, env-driven branch in the canonical entrypoint, or generated
  variant verified in CI).
- Pinned `newrelic` version.
- Explicit `USER` at the end of the Dockerfile.
- Minimal CI job that builds the image.
