# openai/codex #20144 — Fix migrated hook path rewriting

- **PR:** https://github.com/openai/codex/pull/20144
- **Head SHA:** `fd2336e602e8bfd588441ce8a6d35b7b5bea0b5d`
- **Size:** +330 / −85

## Summary
Fixes the external-agent hook config layering so that `.claude/settings.local.json` overrides actually take effect. Previously three hook-related call sites in `external_agent_config.rs` read raw `read_external_settings(...)`, bypassing the precedence resolution that the rest of the config path already used. Switches them to `effective_external_settings(...)`, which composes user → project → local in the documented order.

## Specific observations

1. **Three load-bearing call-site swaps in `codex-rs/cli/src/external_agent_config.rs`:**
   - line 252 (the migration write path)
   - line 569 (the per-tool hook lookup)
   - line 636 (the post-migration verification read)

   All three were calling `read_external_settings(home, project)` and getting back the user-level `~/.claude/settings.json` only. The fix routes them through `effective_external_settings(home, project)`, which already exists and correctly merges `~/.claude/settings.json` ← `<project>/.claude/settings.json` ← `<project>/.claude/settings.local.json`.

2. **Why the bug was invisible to the existing tests:** the prior test suite only set up the user-level file and asserted hooks were detected. There was no test where a `settings.local.json` should have *overridden* a user-level hook — so the precedence violation was a silent no-op for the common case (only user-level config exists). The PR adds tests for the override case, which is the right move.

3. **Migration write path (line 252) is the highest-risk call site** because if the migration reads from the wrong layer, it persists the wrong content into the migrated location and then *that* becomes the new authoritative source. So this PR is not just a "your local override didn't apply" fix — it's also a "your local override will not get clobbered by the migration" fix. Worth calling out in the changelog.

4. **`effective_external_settings` is the canonical accessor used by the runtime hook dispatcher** elsewhere in the codebase. The bug was that `external_agent_config.rs` had its own ad-hoc resolution that pre-dated the canonical helper. Net effect of the PR: removes the parallel resolution path, leaves only one source of truth for "what hooks are effective for this project right now."

5. **+330/−85 is mostly tests.** The production diff is small (the three `read_*` → `effective_*` swaps plus their imports). Most of the additions are scenario tests covering: (a) local overrides user, (b) project overrides user, (c) local overrides project, (d) migration preserves local. This is the right test shape for a precedence bug.

## Risks

- **Behavioral change for anyone who had a `settings.local.json` they didn't realize was being silently ignored.** The fix means those overrides start taking effect on next config load. If a user had stale local overrides from an experiment they forgot about, they'll now apply. Likely net-positive (the file exists for a reason), but a one-line changelog entry is warranted.
- **Migration replay risk:** users who already migrated under the buggy code may have settings persisted from the wrong layer. The PR doesn't add a re-migration step. If post-fix behavior differs from what the user has stored, they'd need to manually edit. Probably acceptable since the migration is one-shot, but worth mentioning.
- The renamed function (`read_external_settings` still exists) means anyone who copy-pasted the old call pattern into a downstream tool will keep hitting the bug. Consider deprecating `read_external_settings` with a doc comment pointing to `effective_external_settings`.

## Suggestions

- **(Recommended)** Add a doc comment on `read_external_settings` saying "Returns user-level only; for layered precedence, use `effective_external_settings`." Otherwise the next person to add a hook call site will hit the same bug.
- **(Recommended)** Changelog entry should call out both the override-restoration behavior and the migration-preservation behavior.
- **(Optional)** Consider whether `read_external_settings` should be made `pub(crate)` or moved to a `_raw` suffix to reduce the surface area for future misuse.

## Verdict: `merge-after-nits`

Correct fix at the right layer. The diff size is dominated by tests, which is exactly what you want for a precedence bug. The only nits are documentation — the production change is mechanical and clearly correct.

## What I learned

When two functions exist for the same data — `read_X` (raw) and `effective_X` (resolved) — and both are publicly callable, the precedence-violation bug class is inevitable. The fix is to either (a) make the raw form `_raw`-suffixed and `pub(crate)`, or (b) make the resolved form the only public path and force callers to opt out explicitly. This PR fixes the symptom but not the call-site footgun.
