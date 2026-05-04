# BerriAI/litellm#27126 — chore(deps): refresh dependency locks

- **Head SHA**: `e96d850b8423`
- **Verdict**: `merge-after-nits`

## Summary

Large lock-refresh PR (+3108/-748). Bumps `cookbook/litellm-ollama-docker-image/requirements.txt` from `litellm==1.83.5` → `1.83.14`, bumps `enterprise/pyproject.toml` build-system requirement `uv_build==0.10.7` → `0.11.8`, and introduces a new `litellm-js/proxy/package-lock.json` (2054 lines) pinning `hono@4.12.16`, `openai@4.29.2`, `wrangler@4.87.0`, and `@cloudflare/workers-types@4.20260501.1`.

## Findings

- `litellm-js/proxy/package-lock.json:1-30` — net-new lockfile. The directory previously had `package.json` only; checking in a v3 lockfile is the right call for reproducible CI builds, but make sure `litellm-js/proxy/package.json` is the source of truth and that the npm/bun command used to generate it is documented somewhere (CONTRIBUTING or a `Makefile` target). Otherwise the next contributor regenerates with a different toolchain and produces a noisy diff.
- `litellm-js/proxy/package-lock.json:69-78` — `@cloudflare/workerd-darwin-64@1.20260430.1` is pinned with `"optional": true` and `os: ["darwin"]`. That's correct for the Cloudflare Workers runtime; just confirm CI matrix actually exercises a Linux build so the Linux variant gets resolved at install time on production.
- `enterprise/pyproject.toml:19` — `uv_build==0.10.7` → `0.11.8` is a minor bump but `uv_build` is a build backend; any release notes around metadata format changes (PEP 621 surface) should be cross-checked. The bump itself is fine; flag in the PR body that this is a build-system change, not a runtime dep.
- `cookbook/litellm-ollama-docker-image/requirements.txt:1` — bumping the cookbook example from `1.83.5` to `1.83.14` is harmless, but since cookbooks are user-facing, the bump should be tied to a feature/fix the example actually exercises. If it's purely CVE/maintenance, fine; if there's a behavior change between those nine patch versions affecting Ollama path handling, mention it.
- The PR title says "refresh dependency locks" but the diff also adds a brand-new lockfile. Title should be `chore(deps): pin litellm-js/proxy lockfile + refresh enterprise build-system` to make the new file discoverable in `git log`.

## Recommendation

Lock refreshes are low-risk by construction. Before merge: (1) document the regeneration command for `litellm-js/proxy/package-lock.json` so subsequent bumps are deterministic, (2) tighten the commit message to surface the new lockfile, (3) confirm CI exercises both darwin and linux workerd binaries. Otherwise looks good.
