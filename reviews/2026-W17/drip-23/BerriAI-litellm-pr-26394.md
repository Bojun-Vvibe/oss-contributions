# BerriAI/litellm PR #26394 — docs(guardrails): add during_call mode to Model Armor guardrail docs

- **Repo:** BerriAI/litellm
- **PR:** [#26394](https://github.com/BerriAI/litellm/pull/26394)
- **Head SHA:** `b79b64b3a90614c9447364f945d8ac512ec2a51b`
- **Size:** +4/-3 in `docs/my-website/docs/proxy/guardrails/model_armor.md`
- **Reviewer:** Bojun (drip-23)

## Summary

Documentation-only PR. Adds `during_call` to the documented set of
supported `mode` values for the Model Armor guardrail integration,
fixes a slightly-misleading description of `post_call` (which the docs
previously said ran on "input & output", now correctly describes as
running on "output"), and updates the YAML config example to include
`during_call` in the `mode:` list.

The runtime apparently already accepts `during_call` for Model Armor
(this is a docs-catch-up PR, not a feature add).

## What's changed

Three edits, all in the same file:

### 1. YAML example (diff lines 27–28 of the source file)

Before:
```yaml
mode: [pre_call, post_call]  # Run on both input and output
```

After:
```yaml
mode: [pre_call, during_call, post_call]  # Run on input, parallel, and output
```

The trailing comment changes from "Run on both input and output" to
"Run on input, parallel, and output". The new comment is more
accurate but parses awkwardly — "input, parallel, and output" reads
like three nouns, but "parallel" is actually describing **when** the
guardrail runs (in parallel with the LLM call), not **what** it
operates on. Suggested rephrase: "Run on input (pre), input in
parallel with the call (during), and output (post)" — clunkier but
unambiguous. Or just: "Run on input and output, with an additional
parallel input check."

### 2. Mode value list (diff lines 41–44)

Before:
```
- `pre_call` Run **before** LLM call, on **input**
- `post_call` Run **after** LLM call, on **input & output**
```

After:
```
- `pre_call` Run **before** LLM call, on **input**
- `during_call` Run **in parallel** with LLM call, on **input**
- `post_call` Run **after** LLM call, on **output**
```

The `post_call` description fix is the most consequential change in
the PR — the old docs said `post_call` validated **both input and
output**, which contradicts the production behavior being asserted
in PR #26388 (the bedrock-guardrails sibling PR I just reviewed):
`should_validate_input = not (... pre_call ... or ... during_call)`
exists *because* `post_call` does in fact validate input *unless*
pre/during already did. So the truth is more nuanced than either
the old or new doc string says:

- pre_call: input only (before)
- during_call: input only (parallel with the call)
- post_call: output **plus** input *if no pre/during is configured*
  (i.e., post_call back-fills input validation).

The new docs are an over-correction — they now imply post_call
validates **only** output, which understates what it actually does
when configured alone. For users running just `mode: [post_call]`
(no pre/during), the input is still validated. The doc should
mention this back-fill behavior explicitly:

> - `post_call` Run **after** LLM call, on **output**. If no
>   `pre_call` or `during_call` is configured, also validates
>   **input** to ensure coverage.

### 3. `mode` parameter description (diff line 78)

Before:
```
- `mode` - Union[str, list[str]] - Mode to run the guardrail. Either
  `pre_call` or `post_call`. Default is `pre_call`.
```

After:
```
- `mode` - Union[str, list[str]] - Mode to run the guardrail.
  Supported values: `pre_call`, `during_call`, `post_call`. Default
  is `pre_call`.
```

Clean. The "Either X or Y" → "Supported values: X, Y, Z" phrasing
is more extensible.

## Concerns

1. **`post_call` description is still wrong, just in a different
   direction**

   See above. The runtime back-fills input validation when only
   `post_call` is configured. That's not documented either before
   or after this PR. Users reading the new docs will assume they
   need to add `pre_call` to get input coverage, when in fact
   adding `pre_call` *removes* the back-fill (since `should_
   validate_input` becomes false). The docs should describe the
   actual policy.

2. **Consistency check — does this language match other guardrail
   docs?**

   The `pre_call` / `during_call` / `post_call` mode taxonomy is
   shared across guardrails in litellm (Bedrock, Aporia, Lakera,
   Presidio, etc.). If the Model Armor doc is now using the
   updated phrasing, the other guardrail docs should too. Otherwise
   a user comparing two integrations gets contradictory
   descriptions of the same `mode` field. A grep across `docs/my-
   website/docs/proxy/guardrails/*.md` for `post_call` would
   confirm whether this PR is one of N or the only one.

3. **No example showing all three modes together**

   The YAML example (line 28) mixes all three modes in one
   `mode:` list. That's the most aggressive configuration; for
   most users a list of one or two suffices. A second YAML example
   showing just `mode: during_call` (the new mode being
   documented) would help users adopt it specifically.

4. **No description of the `during_call` perf trade-off**

   `during_call` runs the guardrail in parallel with the LLM call,
   so it doesn't add latency to first-token-time, but it *does*
   add a second concurrent network request. For users on
   rate-limited providers or with strict spend controls, that's a
   non-trivial cost. The doc could add a one-line note: "Use
   `during_call` to validate input without adding pre-call latency,
   at the cost of an additional parallel API request to Model
   Armor."

## Verdict

`merge-after-nits` —

- correct the `post_call` description to mention the input
  back-fill behavior (or at least link to a behavior-table
  somewhere);
- consider replacing "Run on input, parallel, and output" with
  cleaner phrasing in the YAML comment;
- add a one-line `during_call` perf trade-off note;
- (optional) sweep sibling guardrail docs to apply the same
  three-mode list consistently.

Pure docs PR with low merge risk; the post_call description
issue is a pre-existing inaccuracy that this PR is partly fixing
and partly perpetuating.

## What I learned

The "post_call also back-fills input validation" behavior is the
kind of policy that lives in the code (a single boolean ternary
in `bedrock_guardrails.py`) but is invisible in the docs. Every
litellm guardrail integration has the same `should_validate_
input = not (pre_call or during_call)` pattern, but no doc
mentions it. The right fix is a single shared "Guardrail Mode
Behavior" doc page (or table) that's linked from each individual
guardrail's docs, instead of every guardrail re-documenting the
same modes (and getting them slightly wrong each time). Worth
filing as a doc-restructuring follow-up.
