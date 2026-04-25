# BerriAI/litellm#26474 — fix(bedrock guardrail): dedupe post-call log entry when only post_call is configured

- PR: https://github.com/BerriAI/litellm/pull/26474
- Head SHA: `c5a57b66ab395469457de219816b38b6b86fec6a`
- Base SHA: `70492cee4282541256fb9ac963be94412b1a109c`
- Author: shivamrawat1
- State: OPEN
- Diff: +98 / −108 in `litellm/proxy/guardrails/guardrail_hooks/bedrock_guardrails.py`

## Summary

Replaces an earlier `skip_logging` workaround for a duplicate Bedrock
guardrail log entry with a semantic fix: `async_post_call_success_hook`
and `async_post_call_streaming_iterator_hook` no longer fan-out an extra
`source="INPUT"` Bedrock scan in parallel with the OUTPUT scan. The
duplicate post-call trace entry (e.g. *post-pii* at T+353ms and again at
T+370ms) was a symptom of the hook running INPUT validation under
`event_type=post_call`, which is semantically wrong — input validation
belongs to `pre_call` / `during_call`. Net diff is heavily deletion-leaning,
which is the right direction for this fix.

## Diff highlights

- `bedrock_guardrails.py:1098-1155` — `async_post_call_success_hook` loses
  the `should_validate_input` branch and the
  `await asyncio.gather(input_task, output_task)` call. It now performs a
  single `await self.make_bedrock_api_request(source="OUTPUT", response=…,
  request_data=data, logging_event_type=GuardrailEventHooks.post_call)`
  with a try/except for `GuardrailInterventionNormalStringError`. The
  `import asyncio` line at the top of the function is also removed.
- `bedrock_guardrails.py:1187-1250` — same shape applied to
  `async_post_call_streaming_iterator_hook`: the parallel
  `input_task` / `output_task` `asyncio.gather` is collapsed into one
  OUTPUT-only `make_bedrock_api_request` call, and the docstring is
  updated to match (*"Collect content from the stream and run the bedrock
  OUTPUT scan (post_call only validates the response)."*).
- The previous-commit `skip_logging` workaround on
  `make_bedrock_api_request` is dropped as part of the same PR — preferring
  a real fix over a logging-suppression band-aid is the right call.

## Concerns / Nits

1. **Behaviour change for users running ONLY `post_call`**: prior to this
   PR, configuring only `post_call` was the implicit way to get *both*
   INPUT and OUTPUT scans in a single hook (with the bonus of parallelism
   via `asyncio.gather`). After this PR, those users silently lose
   INPUT validation. The PR description acknowledges this and tells users
   to configure `pre_call` / `during_call`, but: (a) this is a behaviour
   change worth a `BREAKING:` note in CHANGELOG, and (b) consider logging a
   one-shot warning at hook init when only `post_call` is configured,
   pointing users at the migration.
2. **Test coverage**: I'd want to see at least three regression tests:
   (a) only-`post_call` configured emits exactly one trace entry (the one
   that was duplicated before); (b) `pre_call + post_call` configured
   still emits the expected two distinct entries; (c) the streaming hook
   variant matches the non-streaming behaviour. The PR diff doesn't show
   test additions in the snippet — confirm before approving.
3. **Loss of parallelism is fine**: removing `asyncio.gather` cuts one
   round-trip's worth of concurrency, but since post_call now does only
   one request, there's nothing to parallelize. Net latency is *better*
   for only-`post_call` users (one request, not two).
4. **Naming**: now that `should_validate_input` is gone, the dead
   `_event_hook_is_event_type` lookups it was guarding might also be
   unused elsewhere — worth a follow-up grep.

## Verdict

**merge-after-nits** — semantically correct, kills a real bug, replaces a
band-aid with a fix. Block on a CHANGELOG breaking-behaviour note and at
least one regression test asserting the duplicate-log scenario emits one
entry.
