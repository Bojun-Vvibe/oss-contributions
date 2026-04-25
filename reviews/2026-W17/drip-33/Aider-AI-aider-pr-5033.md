# Review: Aider-AI/aider#5033 — Skip auto-commit when LLM response is truncated

- **PR**: https://github.com/Aider-AI/aider/pull/5033
- **State**: OPEN
- **Author**: Fatty911
- **Range**: +28 / −0
- **Head SHA**: `fca90100e3a42a41060e8499a9fa37e990bd1cab`
- **Base SHA**: `f09d70659ae90a0d068c80c288cbb55f2d3c3755`
- **Verdict**: request-changes
- **Reviewer date**: 2026-04-25

## What the PR does

In `aider/coders/base_coder.py`, adds a `response_was_truncated`
flag (reset at the top of `send_message`) that is set in two
places:

1. Inside the `except FinishReasonLength` branch at line ~1492
   when the model does **not** support assistant prefill.
2. After the send loop, in a new heuristic at line ~1550 that
   token-counts `partial_response_content` and trips the flag
   when output is `>= max_output_tokens * 0.92`. Same
   non-prefill gate.

`auto_commit` (line ~2392) checks the flag and, if true,
emits a tool warning and returns early — bypassing the
commit so the user can review.

## Observations

1. **The intent is sound: a truncated edit can be syntactically
   broken, and auto-committing it is a footgun.** Aider's auto-
   commit is one of its strongest UX features but also its
   highest-blast-radius behavior; making it fail-closed on
   suspected truncation is the right policy direction.
2. **The 92% threshold is magic and unjustified.** Why 92%?
   The PR description doesn't motivate it. `max_output_tokens`
   in `model_info` is sometimes a hard limit, sometimes a
   marketing limit, sometimes wrong (Anthropic's 8192 vs
   Claude 3.5's actual ~8192 with prefill). A model legitimately
   producing a 7600-token diff that fills 93% of a documented
   8000-token budget is **not** truncated; it just happened to
   produce a long-but-complete output. The heuristic will
   false-positive on these. Either (a) raise threshold to
   ~99%, (b) require a `finish_reason` *not* explicitly
   "stop"/"end_turn" alongside the token check, or (c) bias
   toward syntactic checks (does the diff end mid-edit-block?
   does the last hunk parse?) which is what this PR really
   wants but is more work.
3. **`token_count(self.partial_response_content)` runs on every
   send_message.** For long responses this is non-trivial CPU
   (tiktoken or equivalent over the full output). Worth gating
   on `len(self.partial_response_content) > some_byte_threshold`
   first to skip the expensive call when the response is
   obviously short.
4. **Reset placement is correct, but the prefill exclusion is
   leaky.** `if not self.main_model.info.get("supports_assistant_prefill")`
   gates both the heuristic *and* the explicit
   `FinishReasonLength` flag-set. That second gate is the
   problem: a prefill-capable model that *does* hit
   `FinishReasonLength` and exhausts its
   `multi_response_content` accumulator can still produce a
   truncated final result. The PR's gate means the flag will
   never be set for prefill models even when they are
   genuinely truncated. The "final check happens later"
   comment in the diff is misleading — there is no later
   check for prefill models.
5. **The `auto_commit` early-return changes the function's
   contract.** `auto_commit` previously returned `None` on
   success-path-not-applicable and a commit hash on success.
   This PR returns a string `msg` ("LLM response was
   truncated. ...") which downstream code may pattern-match
   against `if commit_hash` and silently misinterpret the
   warning string as a successful commit hash. Need to grep
   call sites — at minimum, return `None` and emit the
   warning, don't return the message text.
6. **No tests.** A regression test fixture with a fake
   `FinishReasonLength` raise + assertion that no commit was
   created would lock in the contract. Same for the silent-
   truncation heuristic. Aider has good test infra; this is
   a 30-line test file at most.
7. **`response_was_truncated` is set as instance state but
   never explicitly cleared after `auto_commit` reads it.**
   So if the user manually invokes `/commit` after the
   warning, the flag is still True at the start of the next
   `send_message` (where it gets reset, fine), but if any
   *other* code path reads the flag mid-conversation it sees
   stale state. Minor — initialize cleanly in `__init__` and
   document the lifecycle.

## Verdict reasoning

Right policy ("don't auto-commit suspected partial edits"),
wrong implementation. The 92% heuristic is unjustified, the
prefill-model exclusion is too broad, the `auto_commit`
return-type change has cross-call-site risk, and there are
no tests. Recommend `request-changes`: switch to a
syntactic check (or at minimum, `finish_reason` +
threshold), preserve `auto_commit`'s return contract, and
add 2-3 tests.

## What I learned

"Skip the side effect on suspected bad input" is correct
fail-closed policy, but the *detector* for "suspected bad
input" matters as much as the policy. Token-count thresholds
on legitimate-but-long outputs are exactly the kind of
heuristic that ages badly as model context windows grow.
The robust signal here is structural (does the diff parse
to completion?), not token-arithmetic.
