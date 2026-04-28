---
pr_number: 19990
repo: openai/codex
head_sha: 2edff1624bac845b56596f67152f58446189d27f
verdict: merge-after-nits
date: 2026-04-28
---

# openai/codex#19990 — feat: skip memory startup when rate limits are low

**What changed.** +211/−1 across 10 files. Three layers:

1. **Config surface** at `codex-rs/config/src/types.rs:35,208,228,244,286-289`: new `MemoriesToml.min_rate_limit_remaining_percent: Option<i64>` with `#[schemars(range(min = 0, max = 100))]`, default `25`, clamped to `0..=100` in `From<MemoriesToml> for MemoriesConfig`. Schema regen at `codex-rs/core/config.schema.json:1117-1123`.
2. **Guard module** new file `codex-rs/memories/write/src/guard.rs:1-70`: `rate_limits_ok(auth_manager, config) -> bool` calling `BackendClient::from_auth(...).get_rate_limits_many().await`, finding the snapshot with `limit_id == "codex"` (falling back to `snapshots.first()`), comparing against `config.memories.min_rate_limit_remaining_percent`. Wraps inner `rate_limits_check -> Option<bool>` with `.unwrap_or(true)` so any failure (no auth / non-codex backend / network error / parse error) preserves existing best-effort startup behavior.
3. **Tests** at `codex-rs/config/src/types_tests.rs:62-88` (clamp 101 → 100, -1 → 0) and `codex-rs/core/src/config/config_tests.rs:269,285,310` (TOML round-trip with `min_rate_limit_remaining_percent = 12`).

**Why it matters.** Memory startup runs background LLM calls (extract + consolidation) eligible to consume the same rate-limit budget the user is about to use for the next turn. Hitting the limit during memory startup gives the worst possible UX: the next user turn fails because the guard ran the budget down. This PR adds a pre-flight check that backs off when usage in either window is above the configured ceiling.

**Concerns.**
1. **`unwrap_or(true)` is the right default** (fail-open: preserve current behavior on any error path) and the inner function correctly returns `None` for the `!auth.uses_codex_backend()` case so non-Codex auth keeps unconditionally running memory startup. Good fail-mode shape.
2. **`snapshots.first()` fallback** when no `limit_id == "codex"` snapshot is present is questionable — it silently treats "first arbitrary limit" as the codex limit. If the API ever returns multiple limit kinds without a `codex`-tagged one (e.g. only `chatgpt-plus` and `team`), the behavior is non-deterministic. Either log-warn on the fallback path or make the absence return `None` (skip the check, fail open).
3. **`min_rate_limit_remaining_percent` semantic is "minimum *remaining*"** but the PR body and config comment both phrase it as "usage above the ceiling skips startup" — those are equivalent only when "remaining + used = 100%" exactly. If the rate-limit API reports separate primary/secondary windows where one is reset and the other is exhausted, the helper at `:?-?` needs to take the *minimum* remaining across both windows (which is what the body claims). Worth a unit test pinning that two-window behavior — `guard_tests.rs` is mentioned in the PR body but not visible in this diff slice.
4. **Default `25%` is aggressive** — it means startup is skipped whenever *more than 75% of either window has been consumed*. For users mid-session this is a sensible safety margin, but for bulk eval/CI runs (where rate-limit consumption is steady-state high) it will silently disable memory updates entirely. A `0` value disabling the check is supported (clamped to `0..=100`); document that `0 = always run`.
5. **`status=skipped_rate_limit` counter** is mentioned in the PR body but the metrics-emit site isn't in the visible diff slice; verify the `codex.memory.startup` counter only emits the `skipped_rate_limit` value when the *rate-limit guard* skipped (not when other guards skipped — the `skipped` namespace shouldn't be polluted).
6. **`codex-backend-client` is now a hard dep of `memories/write`** (Cargo.toml `:17`); confirm this doesn't pull a heavy transitive graph into the memory crate that wasn't there before.

Right shape and right defaults. Ship after fixing the silent-fallback at concern 2 and confirming the two-window minimum-of-remaining behavior at concern 3.
