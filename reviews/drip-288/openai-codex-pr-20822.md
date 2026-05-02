# openai/codex PR #20822 — Use structured service tiers in core model metadata

- Head SHA: `05ebe23c94c40b6a08f5a0db7f49a80655da7a3b`
- URL: https://github.com/openai/codex/pull/20822
- Size: +278 / -100, 32 files
- Verdict: **merge-after-nits**

## What changes

This is the *core* counterpart to #20823/#20824. The hard-coded
`ServiceTier { Fast, Flex }` enum in `codex-protocol` is replaced with
a string-id-backed `ServiceTier(String)` newtype, with two well-known
constants `SERVICE_TIER_PRIORITY` and `SERVICE_TIER_FLEX`. The
app-server v2 protocol keeps a *closed* enum `ServiceTier { Fast, Flex }`
and supplies `From<ServiceTier> for CoreServiceTier` (Fast → priority,
Flex → flex) plus a fallible `TryFrom<&CoreServiceTier> for ServiceTier`
so unknown core ids round-trip as `None` rather than panicking.
`build_thread_config_overrides` and `collect_resume_override_mismatches`
are updated to convert via the new `resolve_service_tier_override`
helper.

## What looks good

- The `String`-backed core enum is the right choice for this rollout.
  Hard-coded `Fast`/`Flex` in core was about to block adding any new
  backend service tier (Standard, Batch, etc.) without a synchronized
  protocol bump. The wire-stable `Fast`/`Flex` v2 enum keeps existing
  app-server clients working unchanged — that compatibility statement
  in the PR body matches the diff.
- `TryFrom<&CoreServiceTier> for ServiceTier` returning `Result<_, ()>`
  + `.ok()` at every call site (codex_message_processor.rs lines ~2895,
  ~4445, ~5042, ~8413, ~8589) means a future "standard" tier on the
  backend won't crash a client that only knows `Fast`/`Flex`; it just
  shows up as "no override". That's the right failure mode for this
  protocol surface.
- The `Fast → SERVICE_TIER_PRIORITY` mapping (v2.rs line ~140)
  preserves the PR body's stated "enterprise default = priority"
  convention. `Fast` was previously ambiguous between "OpenAI scale
  fast tier" and "priority"; pinning the string makes it auditable.
- `collect_resume_override_mismatches` (line ~8589) correctly
  re-projects `requested_service_tier.map(CoreServiceTier::from)` for
  comparison against the snapshot, then re-projects active back to v2
  for the diagnostic message. Bidirectional translation in the same
  function is ugly but the *only* way to preserve the existing UX
  string format.

## Nits

1. `TryFrom<&CoreServiceTier> for ServiceTier { type Error = (); }`
   (v2.rs line ~150) — `()` as error type is fine for internal use but
   reads as "we don't care which tier was unknown". A `&str`
   carrying the unknown id would let `Self::resolve_service_tier_override`
   log it. Right now an unknown core tier silently becomes `None` and
   the user sees no signal.
2. The four near-identical `service_tier: ...and_then(|st| ServiceTier::try_from(st).ok())`
   blocks (lines ~2895, ~4445, ~5042, ~8413) cry out for a small
   helper `fn project_service_tier(opt: &Option<CoreServiceTier>) ->
   Option<ServiceTier>`. Same logic, four sites — would have caught
   any future drift.
3. `Self::resolve_service_tier_override(service_tier)` (line ~2972) is
   declared as `fn` on the impl but takes `&self` nowhere — could be a
   free function. Minor, but the `Self::` qualifier suggests state
   ownership that doesn't exist.
4. PR body says "Did not run tests locally; relying on CI per project
   guidance." For a 32-file refactor that touches the resume mismatch
   diagnostic format, *locally running* `cargo test -p
   codex-app-server -- service_tier` once would have been worth the
   60s. The format string at line ~8597 (`"service_tier requested={...}
   active={...}"`) is the kind of thing that has snapshot tests.

## Risk

Medium-low. The compatibility claim is correct *if* every consumer of
`config_snapshot.service_tier` goes through the new `try_from`+`.ok()`
projection. The diff shows 4 such call sites in
`codex_message_processor.rs` and they all use the same pattern, so
that looks complete. Real risk is if any *other* crate reads
`session_configured.service_tier` directly and assumes `ServiceTier::Fast`
or `ServiceTier::Flex` exhaustiveness — that's not visible in this
diff scope but worth a `rg "session_configured.service_tier"` before
merge. Pairs with #20823 (app-server) and #20824 (TUI) — should land
in order: this → 20823 → 20824.
