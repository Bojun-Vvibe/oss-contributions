# opencode PR #24149 — feat(scout): add scout agent for repo research

Link: https://github.com/anomalyco/opencode/pull/24149

## What it changes

Adds a built-in `scout` subagent with two managed tools — `repo_clone`
and `repo_overview` — for dependency-source / external-docs research.
Adds a `util/repository.ts` shared parser, a `util/github-remote.ts`
helper, an app-data cache directory for cloned repos, an entry in the
agent registry, a 36-line scout prompt, permission entries, and
localized `agents.mdx` doc updates across ~15 locales. ~1.2k LOC added,
~50 deleted; bulk of substance lives in `tool/repo_clone.ts` (157 L)
and `tool/repo_overview.ts` (244 L) plus their tests (~350 L combined).

## Critique

This is a feature, not a fix, so the right axes are scope, blast
radius, and how cleanly the new tools compose with the existing
permission/sandbox model.

1. **`repo_clone` is a sandbox-shaped privilege.** Cloning to a
   shared cache under app-data means the scout agent persists
   filesystem state across sessions. The PR registers permission
   entries, which is correct, but the cache itself becomes a
   long-lived attack surface: a previous session's cloned repo can
   later be read/exec-walked by a different agent through
   `repo_overview`. The PR should specify a TTL or per-session
   namespace for the cache, and document that `repo_overview` will
   *only* read from caches the current agent populated. Otherwise
   the cache is silent shared mutable state across agents.
2. **Network-scope of `repo_clone`.** The tool effectively grants a
   network egress + write capability to the scout agent. If the
   user has configured a network proxy / denylist, `repo_clone`
   needs to honor it; otherwise scout becomes a way to bypass
   egress controls. Worth confirming the clone path goes through
   the same HTTP layer that respects user proxy settings.
3. **`repo_overview` is doing a lot.** 244 lines for a single tool
   is on the edge. A separate scan/summary split would let the
   summary side be capped without touching the scan side.
4. **Localization sprawl.** The scout agent surface ships in 15+
   `agents.mdx` locales in the same PR. That is fine for docs, but
   the actual prompt (`scout.txt`) is English-only — so non-English
   users will see translated docs but get an English system prompt.
   The PR should at least call this out.

## Suggestions

- Namespace the cache directory by session ID (or by content hash of
  the requesting agent's prompt) and document the eviction policy.
- Route `repo_clone` through the same HTTP client that honors
  `provider.options.proxy` / network deny rules, and add a test that
  asserts a denied host blocks the clone.
- Split `repo_overview.ts` into `scanner` + `summarizer` modules so
  the summary path can be size-capped independently.

Verdict: useful capability; merge after the cache lifecycle and
proxy-egress story are nailed down.
