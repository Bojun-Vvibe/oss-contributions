---
pr: 4718
repo: browser-use/browser-use
sha: 479ae443a300b996a44c8f560529b618d41429d4
verdict: request-changes
date: 2026-04-26
---

# browser-use/browser-use#4718 — fix: robust Docker detection using multi-signal heuristics

- **URL**: https://github.com/browser-use/browser-use/pull/4718
- **Author**: Myc911
- **Closes**: #4149

## Summary

Replaces the existing `is_running_in_docker()` two-check (`/
.dockerenv` exists, or `'docker' in /proc/1/cgroup`) plus PID-1
cmdline heuristic with a six-step ladder:

1. `/.dockerenv` exists
2. cgroup mentions `docker`
3. `/proc/self/mountinfo` mentions `overlay` or `/docker/containers/`
4. `KUBERNETES_SERVICE_HOST` set, or `container=docker` in env
5. PID-1 cmdline contains py/uv/app/gunicorn/node
6. Total process count < 10

## Reviewable points

- `browser_use/config.py:21-24` — step 1 (`/.dockerenv`) is fine
  but the file doesn't exist on rootless Podman, containerd-only
  setups, or many CI runners. Existing behavior; not a
  regression.

- `browser_use/config.py:26-31` — step 2 (cgroup) is correct on
  cgroup v1 hosts but unreliable on cgroup v2 (the cgroup file
  format changes and may not contain `docker` even when running
  in Docker). The PR doesn't add a v2-aware path; falls through
  to step 3.

- `browser_use/config.py:33-40` — step 3 (mountinfo + `overlay`)
  is **a false-positive trap**. `overlay` is the default
  storage driver for Docker but it's also used by:
  - rootful unprivileged user namespaces on stock Linux
  - `live-build`, `casper`, and most live-CD initramfs
  - `unshare --mount` setups for sandboxing
  - dev-mode systemd-nspawn with `--volatile=overlay`
  Hitting `'overlay' in mountinfo.lower()` returns True on a
  laptop running a `unshare`-based dev sandbox. That's a
  silent regression for people who tune Chrome flags based on
  this detection.

- `browser_use/config.py:42-44` — step 4: `KUBERNETES_SERVICE_HOST`
  is a fine signal *for K8s*, but the function is
  `is_running_in_docker`, not `is_running_in_container`. K8s
  pods use containerd or CRI-O directly on most clusters now;
  Docker shim is deprecated. Either rename the function (better
  but breaking) or scope step 4 to docker-specific signals only.

- `browser_use/config.py:48-52` — step 5 expands the cmdline
  match from `py/uv/app` to `py/uv/app/gunicorn/node`. The
  string `'py'` matches `python`, `pypy`, *and* `mypy`,
  `pytest-running-in-tmux`, etc. — anything containing the
  substring `py`. The original heuristic had this issue; the PR
  inherits and amplifies it (adding `'node'` matches `nodejs`
  but also any binary with `node` substring like
  `nodemon-running-locally`).

- `browser_use/config.py:55-57` — step 6 (PID count < 10)
  remained unchanged from the original. Correct as a final
  fallback but it can fire on freshly-booted minimal VMs and
  on `docker-machine`-style intermediates.

- The function is `@cache`'d, so all six checks run exactly
  once per process. No perf concern.

- No tests. For a heuristic ladder where each step has a
  documented false-positive scenario, locking the expected
  return values for at least the four "definitely Docker"
  cases (.dockerenv present, cgroup match, mountinfo
  `/docker/containers/` match, env=docker) is the bare
  minimum. Currently the function has zero unit tests in the
  repo (verified via grep).

## Rationale

The intent (more signals → fewer false negatives) is right, but
the implementation makes the false-*positive* rate strictly
worse: step 3 (`'overlay' in mountinfo`) will fire on dev
laptops using user-namespace sandboxes, and step 4 conflates
"in Kubernetes" with "in Docker." For a function that gates
Chrome launch flags (dev-shm sizing, GPU settings), false
positives mean wrong sandbox tuning on developer machines —
silently degrading interactive performance. Tighten step 3 to
`'/docker/containers/' in mountinfo` only (drop the `overlay`
arm), drop or rename for step 4, and add the regression tests.

## What I learned

"Robust container detection" is one of those problems where
adding more heuristics can make the function strictly worse if
each new heuristic has a higher false-positive rate than the
existing ones. The right move is usually to *narrow* the most
specific signals (e.g. `/proc/1/cgroup` matching the literal
substring `/docker/`, not just `docker`) before reaching for
broader signals like overlay-FS or environment variables.
