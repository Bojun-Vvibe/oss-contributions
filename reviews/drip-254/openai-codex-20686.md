# openai/codex #20686 — test: make async cancellation test deterministic

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20686
- **HEAD SHA:** `5ba42268e892f4a10531a3eb2725ff20d2dea620`
- **Author:** bolinfest
- **Verdict:** `merge-as-is`

## What the diff does

Two-line change inside `codex-rs/async-utils/src/lib.rs:34-66`
(`returns_err_when_token_cancelled_first`): replaces the `async {
sleep(Duration::from_millis(100)).await; 7 }` work future with
`std::future::pending::<i32>()`, keeping the spawned 50ms-delay
cancellation task as-is.

Diff is +2 / -6:

```rust
- let result = async {
-     sleep(Duration::from_millis(100)).await;
-     7
- }
- .or_cancel(&token)
- .await;
+ let result = pending::<i32>().or_cancel(&token).await;
```

plus `use std::future::pending;`.

## Why it's right

The test's *intent* is "if the token is cancelled before the work
future resolves, `or_cancel` returns `Err(CancelErr::Cancelled)`."
The prior implementation used a 100ms sleep as the work future and a
50ms-delayed cancellation, relying on the 50ms gap to make
cancellation win the race. That worked on a quiet developer machine
but is exactly the kind of timing-dependent test that goes flaky on
loaded CI runners (GitHub Actions, container builds, anything with
co-tenant noise) — a 60ms scheduler hiccup on the cancel task and
the 100ms sleep wins the race instead, the work future returns `Ok(
7)`, and the assertion `assert_eq!(Err(CancelErr::Cancelled),
result)` fails for reasons unrelated to the code under test.

`std::future::pending()` is the canonical "future that never
resolves" primitive — by definition it cannot win any race against
any other future. So the test now asserts the *exact* property it
means to assert: "given a work future that never resolves, the
cancellation token resolution path must be the one that fires." The
50ms delay on the cancellation task is preserved, so the test still
exercises the *asynchronous* cancellation code path (token cancelled
during the `select!` poll, not before the `or_cancel` call) rather
than degrading to "token already cancelled at the call site"
(which would test a different branch of `or_cancel` entirely — the
fast-path early-return rather than the wait-and-select path).

`pending::<i32>()` (turbofish on the type parameter) is needed
because `or_cancel` is generic over the work future's `Output` type
and the prior `async { ...; 7 }` block resolved to `i32` via type
inference; the turbofish keeps the inferred output type the same so
no other generic parameters in the test signature shift.

This is a textbook "remove the timing dependency by replacing the
slow side of the race with a never-resolving primitive" cleanup.
Two lines, perfectly scoped, no behavior change to production code,
test surface narrowed to the exact property under test.

## Nits

None worth blocking. The change is minimal, the comment in the PR
body ("Keep the delayed cancellation task so the test still
exercises asynchronous cancellation rather than an already-cancelled
token") names the load-bearing detail (why preserve the 50ms delay
even when the work future is `pending`) for the next reader. A
matching comment *in the test source* at `:62` would be marginally
nicer ("// `pending::<i32>()` makes the work side of the race
unambiguously lose; the 50ms cancellation delay still keeps this
exercising the async cancellation path rather than the fast-path
early-return") but is not load-bearing.

## Verdict rationale

Right diagnosis (CI-flake from timing-dependent race), right
primitive (`std::future::pending` to make the work side unambiguous-
ly lose), preserves the asynchronous-cancellation code path the
test means to exercise (50ms delay on the cancel task retained), no
production code change, two-line scope. Single-author PR from a
maintainer with the test-pinning intent named clearly in the
description.

`merge-as-is`
