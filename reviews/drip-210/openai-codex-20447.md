# openai/codex#20447 — Fix MCP status/list lifecycle leak

- PR: https://github.com/openai/codex/pull/20447
- Head SHA: `64faf26d8c06a6b74acca50881d93f14a0e6bc1d`
- Author: jif-oai
- Files: 4 changed, +275/−54

## Context

`mcpServerStatus/list` is declared as a *globally-serialized* RPC on the
app-server: the framework wraps each request future in a connection-RPC gate
so two list requests can't run concurrently. The handler in
`codex_message_processor.rs` was breaking that contract by `tokio::spawn`-ing
the actual inventory work into a detached task and immediately returning, so
two overlapping `list` requests would each spawn a fresh
`McpConnectionManager`, start MCP clients in parallel, and (per the PR
description) leak processes that the on-completion shutdown logic never saw.
The same detached-task pattern existed for `mcpServer/resource/read` and
`mcpServer/tool/call`.

## Design

The fix is the smallest possible — three call sites in
`codex_message_processor.rs` lose their `tokio::spawn { ... }` wrapper and
become direct `.await` calls in the request future:

- `list_mcp_server_status` at `codex_message_processor.rs:5599-5611` — the
  spawn around `Self::list_mcp_server_status_task(...)` is deleted; the
  future's seven captured args (`outgoing`, `request`, `params`, `config`,
  `mcp_config`, `auth`, `runtime_environment`) flow through unchanged.
- `mcp_resource_read` (the threadless branch at `:5743` and the
  `read_mcp_resource_without_thread` branch at `:5770-5785`) — both lose the
  detached task and `.await` the read inline.
- `mcp_tool_call` at `:5818-5826` — same treatment; the `outgoing.send_result`
  call is now part of the same future the framework gate is holding open.

A trivial cleanup at `:3604` drops a `.clone()` on `request_id` in the
rollback path now that the value is consumed once.

## Verification — the test is the interesting part

`tests/suite/v2/mcp_server_status.rs:226-296` introduces
`BlockingInventoryServer` + `InventoryConcurrencyTracker` and writes
`mcp_server_status_list_serializes_inventory_work` (`:357-440`).

The test:
1. Stands up an MCP server whose `list_resources` blocks on
   `tracker.release_resource_calls` (`:283-285`) and bumps both a high-water
   `max_resource_calls` and a running `active_resource_calls`.
2. Fires request 1, waits until `started_resource_calls == 1`.
3. Fires request 2, then **asserts that `wait_for_resource_call_count(&tracker, 2)`
   times out within 750 ms** (`:401-410`) and that
   `tracker.max_resource_calls.load(Acquire) == 1` (`:411`).

This is the right shape: instead of asserting "there's no leak" via
process-counting (flaky), it asserts the upstream invariant — that the
framework's serialization gate is actually covering the inventory work. If a
future refactor reintroduces a `tokio::spawn` here, this test fails
immediately and locally.

The 750 ms window is the only thing I'd want to revisit in CI; on a loaded
runner this is tight. A `Duration::from_secs(2)` would be safer with no loss
of signal.

## Risks

- `list_mcp_server_status_task`, `read_mcp_resource_without_thread`, and the
  thread-bound `read_mcp_resource` calls now run inside the framework's
  serialization gate. If any of those internally `await` a long network call
  (which they do — that's the whole point of the inventory work), the gate
  is held for the full duration. The PR is explicit that this is the
  intended behavior — the previous "fix" of detaching the work was the bug.
  Worth a one-line comment at the call site documenting "MUST stay
  in-future; see #20447" so the next person to "optimize" this doesn't
  reintroduce the spawn.

- The `stdio_server_launcher.rs:32-44` hardening (surface termination
  failures, keep the process handle retryable, report `taskkill` exit on
  Windows) is a separate concern from the lifecycle leak but lives in the
  same PR. It's the right change but slightly stretches the PR scope —
  splitting it would make the lifecycle fix a one-commit revert if needed.

## Verdict

**`merge-as-is`**

The diff is exactly the size of the bug. The regression test asserts the
right invariant (concurrency upper bound under the framework gate, not
process-count side-effects). The Windows/stdio hardening is in-scope enough
to not block — it's the same lifecycle-correctness theme. The 750 ms timeout
and the documenting comment are nice-to-haves, not blockers.

## What I learned

When a framework provides a serialization gate, the *handler must keep its
work inside the future the gate is holding* — `tokio::spawn` from inside a
gated handler is a covert escape hatch that nullifies the gate. This is the
second time this drip-batch I've seen a "spawn-and-return-immediately"
pattern silently violate an upstream invariant (the first being the
fork-handler in goose's recipe-discovery path). The right test shape is also
the one this PR uses: instrument the *server side* (or a fake of it) with an
`AtomicUsize` high-water mark, then assert the gate keeps that mark at 1.
This generalizes — any time a system claims to serialize work, the
high-water-mark fake-server test is a 30-line investment that catches every
future regression.
