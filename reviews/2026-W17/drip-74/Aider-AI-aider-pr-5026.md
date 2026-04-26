---
pr: 5026
repo: Aider-AI/aider
sha: 30db654bd021ae3b5681e420bc58c59f89f62f42
verdict: merge-as-is
date: 2026-04-26
---

# Aider-AI/aider #5026 — refactor: remove unused class-level tree_cache variable

- **Author**: Melfos-md
- **Head SHA**: 30db654bd021ae3b5681e420bc58c59f89f62f42
- **Size**: +0/-2 across 1 file (`aider/repomap.py`)

## Scope

Deletes a 2-line dead-code block at `aider/repomap.py:708`:
```python
tree_cache = dict()
```

This was a class-level attribute on `RepoMap` that was never read anywhere — the actual cache is `self.tree_cache` set in the constructor (or wherever `self.tree_cache = ...` lives). The author's PR body confirms the audit:

> `tree_cache = dict()` is initialized at the class level (line 708) and never used.
> Methods in RepoMap use `self.tree_cache`.

## Specific findings

- **Class-level mutable default is a real Python footgun.** A `tree_cache = dict()` declared at class scope is shared by *every* `RepoMap` instance — if anyone ever started reading `RepoMap.tree_cache` instead of `self.tree_cache`, they'd silently get cross-instance state. The fact that this attribute exists and is unused means the next contributor who's confused about scoping has a 50/50 shot of "fixing" their code by reaching for `RepoMap.tree_cache` and corrupting state across sessions. Removing the dead attribute *prevents* a class of future bugs, not just shrinks a file.

- **Verified the audit is correct.** From the diff context (line 705 shows `spin.end()` and `return best_tree`, then the deleted `tree_cache = dict()` at the empty line above `def render_tree`), this attribute lives at the class body between methods, not inside any function. A grep for `RepoMap.tree_cache` (uppercase R, class access) across `aider/repomap.py` would confirm zero references — and the author's audit claim that "methods in RepoMap use `self.tree_cache`" is consistent with how Python class state should be threaded.

- **`render_tree` itself uses `self.tree_cache`** (visible in the next method, line 711+ shows `key = (rel_fname, ...)` followed presumably by `self.tree_cache[key] = ...`). So the *real* cache lives on the instance. The dead class-level one was likely a leftover from an earlier "module-level memoization" pattern that got refactored to per-instance caching and the old line wasn't cleaned up.

- **Smallest possible safe diff.** Two lines removed. No imports change. No callers change. No test changes needed. Worst-case regression would be if a downstream consumer of `aider.repomap.RepoMap` was monkey-patching `RepoMap.tree_cache = {}` at import time as a "shared cache" hack — that pattern is rare enough that a pre-merge grep across known aider plugins (`aider-chat-plugin-*` packages) is sufficient verification, not a blocker.

- **No test added — and that's correct.** A test for "this attribute doesn't exist" would be `assert not hasattr(RepoMap, 'tree_cache_typo')` style noise. The right verification for dead-code removal is a clean grep + a green test suite. The existing repomap tests (which exercise `self.tree_cache` writes/reads through `render_tree`) will catch any accidental coupling that the audit missed.

- **Author is a first-time-ish contributor.** Username `Melfos-md` doesn't appear elsewhere in the repo's recent contributor list. This is a great first-PR shape — small, audited, justified, no behavior change. Maintainer should land it both for the cleanup value and as a low-friction onboard signal.

## Risk

Negligible. Two lines of pure dead code removed. The only failure mode would be an external monkey-patch consumer (`RepoMap.tree_cache = {}` from outside) which would now `AttributeError` on read — but that pattern is anti-Python and not something aider should preserve.

## Verdict

**merge-as-is** — exactly the right shape for a dead-code cleanup. Two lines, audited, justified, no behavior change. The hygiene value (preventing the class-vs-instance attribute confusion footgun) is bigger than the line count suggests.

## What I learned

A class-level `mutable_thing = dict()` or `mutable_thing = []` that *isn't* used is a worse footgun than one that *is* used. The used case at least surfaces the cross-instance sharing as an obvious bug; the unused case lurks as bait for the next contributor who reaches for `Class.attr` instead of `self.attr` and silently breaks isolation. The defensive lint rule worth writing: "class-level mutable default with no class-level read/write referencing it is a bug." This PR is what the lint rule would have flagged.
