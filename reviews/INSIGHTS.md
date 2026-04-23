# Cross-Cutting Insights

After reviewing 24 PRs across 8 OSS AI-coding-agent projects (aider,
opencode, OpenHands, litellm, MCP servers, codex, cline, crush), the
same five structural patterns showed up over and over. They are worth
naming because they generalize beyond any one project.

## 1. The send-vs-store boundary is the most important interface

LLM agents send the *full conversation history* with every request,
because that's how the model preserves context. This makes any per-
request constraint a *cumulative* session-wide constraint. The first
session-deadlock bug looks like "Gemini broke" or "Anthropic broke";
the actual diagnosis is always "we hit a per-request quota
cumulatively, and now every subsequent request inherits the failure".

The right architectural move is the *send-vs-store split*. Persist
the canonical history in a durable store; produce a *derived view*
just before sending; mangle the derived view freely to fit whatever
the current provider demands.

- **crush #2613** (image pruning) is the textbook case: prune the
  oldest images from the sent payload, leave the DB untouched, replace
  with a placeholder so the model knows what was removed.
- **aider #4858** (double-buffer context) does the same thing for
  token count: maintain two parallel histories and swap when one
  overflows.
- **litellm #26285** (preserve reasoning content) is the inverse —
  don't *strip* something from the sent payload that the next provider
  needs.

Once you have the send-vs-store split, every future provider quirk is
a one-line transform in the send path. Without it, every quirk is a
breaking change to your data model.

## 2. Pubsub fan-out beats single-channel every time you add a second consumer

The bug pattern: a system has one consumer of an event stream, so it
uses `chan T`. A second consumer is added. Both consumers `range` over
the same channel. Every event goes to whichever reads first. The
"fix" is a send-with-timeout that drops messages, plus a slow-consumer
warning, plus a deadlock waiting to happen.

The actual fix is replacing the channel with a fan-out broker. Each
subscriber gets its own channel; every event is delivered to every
subscriber.

- **crush #2663** is the exemplar: 20 lines of timer juggling and
  drop-on-full collapse to one `Publish` call.
- **crush #2579** (ask-user-questions) reuses the same broker for the
  request-reply pattern, which is the second canonical use of a fan-
  out broker (one publisher, one *eventual* replier, many silent
  observers).

The diagnostic: when you find yourself writing send-with-timeout-and-
drop on a single channel, you are doing fan-out with the wrong tool.
Reach for the broker.

## 3. The tool description prompt is a user-facing API

The model talks to your system through tool descriptions. Treat them
as code. The best tool descriptions in this batch follow a consistent
structure: usage, when-to-use, when-NOT-to-use, conventions, limitations,
result shape.

- **crush #2579** (`ask_user_questions.md`) is the gold standard.
  It includes the "(Recommended)" suffix convention, the "None of the
  above" escape hatch, *and* a "do not ask for feedback while working
  on a plan the user can't see" guardrail that obviously came from
  observing a real failure mode.
- **codex #19058** (auto-review-denials) and **MCP servers #3890**
  (compare_directories) both win by clearly enumerating *when not to
  use* the tool — negative space matters as much as positive.

The general rule: every failure mode you observe in production should
become a `<limitations>` or `<when_not_to_use>` line in the tool
description. It is the cheapest, fastest fix.

## 4. Authorization centralization beats per-route checks

Multi-route auth is a class of bug that recurs because the natural
shape — "add a permission check inside each handler" — is exactly the
shape that lets new handlers ship without checks. Centralization
(decorator, middleware, gateway) makes "no check" the syntactically
visible default.

- **litellm #26279** is a direct example: pull `common_checks` out
  of every route and into a single chokepoint. The bypass disappears.
- **litellm #26274** (MCP OAuth hardening) does the same for OAuth
  endpoints: one verified discovery point instead of several
  diverging ones.
- **codex #18868** (MITM hooks for HTTPS clamping) generalizes the
  pattern beyond auth: any per-host policy belongs at one chokepoint,
  not scattered through callers.

If you ever review a PR that adds a permission check to "this new
handler", ask whether the project has a central auth layer. If not,
adding *another* per-handler check makes the eventual centralization
harder.

## 5. Wrap synchronous libraries with the IO-swap + `run_coroutine_threadsafe` pattern

When you wrap a synchronous CLI library behind an async server, three
things matter: how you intercept the library's output, how you bridge
threads, and how you maintain per-session isolation.

- **aider #4936** (ACP adapter) is the exemplar: swap `coder.io` for
  an ACP-aware subclass, run `coder.run()` in a thread pool, restore
  the IO in `finally`. Cross-thread send via
  `asyncio.run_coroutine_threadsafe` with per-call `future.result(
  timeout=5)`.
- **OpenHands #13983** (ACP routing) is the host-side counterpart —
  same protocol, opposite side.

The IO-swap pattern is reusable for any library with a single output
sink-on-`self`. The cross-thread bridge is the canonical "sync code +
async transport" idiom and worth memorizing. The per-session
invariant (one `Coder` per session, never shared) is the foot-gun
that wraps eventually step on; document it loudly.

---

## Honourable mention: opt-in flag with legacy fallback

Several PRs (cline #10329 cost-control, crush #2613 image pruning,
opencode #23866 worktree paths, MCP servers #3959 memory read modes)
all use the same shipping pattern: new behaviour gated by a config
flag that defaults to legacy. Lets you ship without breaking anyone,
gather telemetry on the new path, and flip the default when confidence
is high. This is just *good operational discipline*, not a structural
insight, but it's worth noting how universal it is across mature
projects.
