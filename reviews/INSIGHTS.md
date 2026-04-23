# Cross-Cutting Insights

After 48 PR reviews across 8 OSS AI-coding-agent projects (aider,
opencode, OpenHands, litellm, MCP servers, codex, cline, crush),
the same structural patterns recur. The first five came out of
the original 24 (W7); patterns 6–10 emerged from the W9 batch,
which leaned heavily into subtle correctness bugs.

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
- **opencode #23870** (session compaction) is the same pattern at the
  compaction layer: derive the compacted view, never destroy the
  source-of-truth history.

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
  request-reply pattern.

## 3. The tool description prompt is a user-facing API

The model talks to your system through tool descriptions. Treat them
as code. The best tool descriptions in this batch follow a consistent
structure: usage, when-to-use, when-NOT-to-use, conventions, limitations,
result shape.

- **crush #2579** (`ask_user_questions.md`) is the gold standard.
- **codex #19058** (auto-review-denials) and **MCP servers #3890**
  (compare_directories) both win by clearly enumerating *when not to
  use* the tool.

The general rule: every failure mode you observe in production should
become a `<limitations>` or `<when_not_to_use>` line.

## 4. Authorization centralization beats per-route checks

Multi-route auth is a class of bug that recurs because the natural
shape — "add a permission check inside each handler" — is exactly the
shape that lets new handlers ship without checks.

- **litellm #26279** is a direct example: pull `common_checks` out
  of every route and into a single chokepoint.
- **litellm #26274** (MCP OAuth hardening) does the same for OAuth
  endpoints.
- **codex #18868** (MITM hooks for HTTPS clamping) generalizes the
  pattern beyond auth.

## 5. Wrap synchronous libraries with the IO-swap + `run_coroutine_threadsafe` pattern

When you wrap a synchronous CLI library behind an async server, three
things matter: how you intercept the library's output, how you bridge
threads, and how you maintain per-session isolation.

- **aider #4936** (ACP adapter) is the exemplar: swap `coder.io` for
  an ACP-aware subclass, run `coder.run()` in a thread pool, restore
  the IO in `finally`.
- **OpenHands #13983** (ACP routing) is the host-side counterpart.

---

## 6. Implicit ambient state is the single highest-leverage correctness bug

A constructor that "conveniently" reads a value from process global
state (`os.getcwd()`, `time.now()`, `os.environ`, the active git
branch) silently couples a request-scoped decision to a process-scoped
value. It looks like ergonomics; it is hidden coupling. The
fix is structural: delete the convenience constructor, force every
caller to pass the value, let the type system enumerate the call sites.

- **codex #19046** (exec-server require explicit cwd) deletes
  `FileSystemSandboxContext::new(SandboxPolicy)` so callers can no
  longer accidentally synthesize a cwd from `current_dir()`. Net-zero
  LOC, large reduction in attack surface.
- **codex #19031** (relative stdio MCP cwd fallback) is the inverse
  symptom — the launcher *should* have read an explicit fallback cwd
  but was implicitly using the host process cwd, causing CLI/Electron
  parity divergence.

The general rule: any time you see `Foo::new(X)` where `Foo` actually
needs `X` *and* `Y`, but `Y` has a "sensible default" pulled from
process state, that constructor is a latent bug. Add the parameter
explicitly.

## 7. Silent payload drops require a structured "no-op" log

The most frustrating bug class in this batch: an HTTP handler
silently discards the request body because of a subtle wire-format
mismatch (frontend sends `agent_settings`, backend expects
`agent_settings_diff`; pydantic discards unknown keys; `has_updates()`
returns `False`; service short-circuits; status: 200). Users
report "save does nothing" — invisible to the operator, invisible
to the logs, invisible to the metrics. Months pass before someone
diagnoses it.

- **OpenHands #14044** is the textbook case: four bugs compounded,
  the first of which was a silent payload drop that made the other
  three impossible to notice.

The fix has two parts: (a) align the wire format; (b) install a
structured log on every "no-op" branch of a write endpoint. If
`has_updates()` returns `False` on a `POST /update`, that is *always*
worth one log line. The cumulative debugging time saved is enormous.

## 8. Symmetric invariants need symmetric tests

When a system has an invariant that ties two halves of a structure
together — every `tool_use` has a matching `tool_result`, every
`open` has a matching `close`, every `acquire` has a matching
`release` — fixing one direction without testing the other guarantees
the next regression is in the other direction.

- **crush #2622** (orphan `tool_use` → inject synthetic result)
  and **crush #2615** (orphan `tool_result` → drop) are the two
  halves of the Anthropic pairing invariant. They landed
  separately. Each has its own tests. There is *no* test that
  asserts the combined property: "after validation, every
  `tool_use` has a matching `tool_result` and vice versa."
- **codex #19086** (filesystem entries) is similar: legacy
  `read`/`write` and v2 `entries` must stay consistent in both
  directions. The PR fixes the missing direction; the symmetric
  property test would lock in both.

The general rule: the moment you write the second half of an
invariant fix, also write the property test that asserts the
invariant. Costs 10 lines, eliminates the bug class.

## 9. Two surfaces sharing one backend = parity bugs in latent

When CLI and IDE share an `rmcp-client`, when frontend and backend
share a constants file, when desktop and web share a session
schema — the divergence point is always "one surface hands the
backend slightly different ambient context, the backend reads
ambient context instead of explicit parameters, parity breaks."

- **codex #19031** (CLI vs Electron stdio MCP cwd) is the textbook
  case: same launcher code, different host environment, silent
  divergence on relative paths.
- **OpenHands #14042** (org-defaults redirect loop) is parity
  between two route guards: each enforces its own redirect
  policy, neither knows about the other, the cycle is not
  visible to either.

The fix is the same as pattern 6: push the ambient dependency up
into an explicit parameter at the boundary. The diagnostic is:
"if frontend A and frontend B see different behavior from the
same backend, the backend is reading something from ambient
context that should be a parameter."

## 10. Restoration deserves heavier tests than the original

When a "remove dead code" PR deletes a feature that was still
being consumed via a now-orphaned settings toggle, the
restoration is not "put it back." It's "put it back, integrated
with the *current* state architecture, and write enough tests
that the next dead-code sweep fails loudly."

- **OpenHands #14049** (notification sound restore) does this
  right: 165 LOC of restored code, 259 LOC of tests (1.5×). The
  hook is rewritten against the V1 agent-state system, not
  ported verbatim against the legacy state model. The bug class
  ("settings toggle bound to nothing") is permanently retired.

The general rule: restoration is a structural opportunity. Don't
re-introduce the original architecture; re-implement against
current architecture and over-test.

---

## Honourable mention: opt-in flag with legacy fallback

Several PRs (cline #10329 cost-control, crush #2613 image pruning,
opencode #23866 worktree paths, MCP servers #3959 memory read modes)
all use the same shipping pattern: new behaviour gated by a config
flag that defaults to legacy. Lets you ship without breaking anyone,
gather telemetry on the new path, and flip the default when confidence
is high.

## Honourable mention: deprecation by description, not by removal

Schema migrations during the deprecation window must keep old
fields populated alongside new ones. The deprecation lives in
the schema *description* (and optionally a `#[deprecated]` Rust
attribute, a JSDoc tag, or a linter rule), not in a sudden
"stop populating the old field." codex #19086 is the example —
both `read`/`write` and `entries` are emitted; the description
text marks the legacy fields for removal in a future version.
