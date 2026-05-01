---
pr: block/goose#8787
title: "refactor: make ACP server smaller"
head_sha: 3b917db5625aedf18414e81362709ef3602f2f44
verdict: needs-discussion
drip: drip-227
---

## Context

`crates/goose/src/acp/server.rs` had grown to ~3,000 lines with two
dominant sources of repetition: (a) the `.map_err(|e|
sacp::Error::internal_error().data(format!("...: {e}")))` boilerplate
appearing dozens of times after every fallible call, and (b)
`debug!(target: "perf", sid = %sid, ms = t_step.elapsed().as_millis() as
u64, ...)` timing spans bracketing every internal step. This PR
introduces a `ResultExt` extension trait to compress the error pattern
and rips out all `target: "perf"` instrumentation, netting -208 lines
in a single file.

## Diff walkthrough

The new `ResultExt<T>` trait at `acp/server.rs:84-107` provides four
methods on any `Result<T, E: Display>`:

- `internal_err()` → `internal_error().data(e.to_string())`
- `invalid_params_err()` → `invalid_params().data(e.to_string())`
- `internal_err_ctx(ctx)` → `internal_error().data(format!("{ctx}: {e}"))`
- `invalid_params_err_ctx(ctx)` → ditto with `invalid_params()`

The `#[allow(dead_code)]` attribute on the trait at `:88` covers the
`internal_err()` and `invalid_params_err()` no-context variants, which
the diff doesn't actually call (every site uses `_ctx` form). A
follow-up could either delete the unused arms or add call sites.

A small helper at `:795-797` adds `fn config(&self) -> Result<Config,
sacp::Error>` wrapping `self.load_config().internal_err_ctx("Failed to
read config")` — replacing the 4-line `.map_err(|e| ...)` block at
every config read site.

The `set_model` rewrite at `:2303-2339` is the cleanest example of the
trait in action: nine `.map_err(|e| sacp::Error::internal_error()
.data(format!("...: {}", e)))` blocks collapse to nine
`.internal_err_ctx("...")` calls, with all the `t_step =
Instant::now()` / `debug!(target: "perf" ...)` instrumentation
removed in the same hunks.

The PR body says: *"I wasn't sure if anyone was using all the perf logs
so I ripped them out, but feel free to let me know if that is wrong"*
— this is the discussion point. The `target: "perf"` filter was a
real, named tracing namespace presumably consumed by an external
observer (maintainer dashboards, perf-regression CI gates, customer
support tooling). Deleting it unilaterally without a migration path is
the load-bearing risk.

## Risks / nits

- **No replacement for the `perf` target**. If any operator or
  developer was filtering with `RUST_LOG=...,goose::acp::server[perf]=debug`,
  they now get silence. A grep for `target: "perf"` outside this file
  would tell us if there's a documented contract; the diff shows none
  in this repo. **Needs author confirmation that no internal dashboard
  consumed these.**
- The trait is added at `crate-private` scope (`trait ResultExt`,
  no `pub`); good — contains the API surface to this file. But
  `#[allow(dead_code)]` masks the fact that two of the four methods
  are unused at merge time. Either delete or use.
- The `set_model` example replaces 9 timed steps with no
  instrumentation. If `set_model` becomes slow in production, the
  granular `t_step` spans are gone and the only signal is the outer
  `t_total` — also gone. Recommend at minimum keeping the outer
  `t_total` debug at each function entry/exit so a regression in
  *which* operation got slow can be triangulated.
- The error context strings are now string literals (`"Failed to read
  config"`) rather than format strings — good, but a few sites lose
  dynamic context (e.g. `provider_name` was previously included in
  some `target: "perf"` debugs and is no longer emitted anywhere).
- The `agent_setup` background-task removal at `:931-1013` deletes
  every `t_setup`, `t_prov`, etc. timing in the session-bootstrap
  hot path. This is the operationally-load-bearing path — slow
  session bootstrap is a top user complaint. Worth keeping at
  minimum a single end-to-end timing.

## Verdict

**needs-discussion** — the trait introduction is clean and clearly
beneficial; ship that part as-is. The `target: "perf"` removal is
the open question. Either (a) author confirms no internal consumer
and adds a one-line release note, (b) `target: "perf"` debugs are
preserved at function-entry/exit granularity (drop the inner steps,
keep the totals), or (c) the perf logs are migrated to a structured
metric. As written, this is a one-way cleanup that loses
production-debugging signal with no obvious replacement.
