# Review: google-gemini/gemini-cli#26266 — Fix posting invalid response to a comment

- PR: https://github.com/google-gemini/gemini-cli/pull/26266
- Merge SHA: `0af13141b235a192f0b361884273b7b72eb81e4c`
- Files: `.github/workflows/gemini-cli-bot-brain.yml` (+13/-2), `tools/gemini-cli-bot/brain/common.md` (+9/-9), `tools/gemini-cli-bot/brain/interactive.md` (+10/-6)
- Verdict: **merge-as-is**

## What it does

Two coordinated fixes to the Gemini CLI bot's CI workflow that runs in this
repo to triage issues/PRs:

1. **Fallback comment when the agent silently fails.** Adds a post-run guard
   at `gemini-cli-bot-brain.yml:143-149` that detects the case where the
   workflow was triggered by a comment (`TRIGGER_ISSUE_NUMBER` set) but the
   agent produced neither `issue-comment.md` nor `pr-comment.md`. In that
   case it writes a generic `"Gemini CLI Bot failed to generate a response"`
   message linking back to the Actions run log, so the comment-author isn't
   left waiting in silence.

2. **Tighten brain prompts to require `write_file` calls.** Edits
   `brain/common.md` and `brain/interactive.md` to change every "write your
   response to `issue-comment.md`" instruction into "use the `write_file`
   tool to save your response to `issue-comment.md`." The strongest of these
   adds a load-bearing `MUST use the write_file tool ... DO NOT simply
   output your response to the console` directive at `interactive.md:69-73`
   with the explicit rationale: "The workflow relies on `issue-comment.md`
   being created in the workspace to post the comment."

3. **Selective brain-state restore.** Replaces a blanket `gh run download
   "$LAST_RUN_ID" -n brain-data -D .` with a temp-dir extract + selective
   `cp` of only `lessons-learned.md` and `history/*.csv` at
   `gemini-cli-bot-brain.yml:88-96`. Prior behaviour restored *every* file
   from the artifact, including transient files like `pr-comment.md`,
   `pr-number.txt`, `issue-comment.md` from the previous run — which were
   then re-posted on the next trigger.

## Notable changes

- `gemini-cli-bot-brain.yml:88-96` — `mkdir -p .temp_brain_data && gh run
  download ... -D .temp_brain_data` then explicit `cp .temp_brain_data/.../
  lessons-learned.md ...` and `cp .temp_brain_data/.../history/*.csv
  history/`. The `2>/dev/null || true` swallows missing-file errors so a
  fresh project (no prior `lessons-learned.md`) doesn't fail the step. The
  `rm -rf .temp_brain_data` cleanup at the end avoids artifact bloat in the
  next upload.
- `gemini-cli-bot-brain.yml:143-149` — `[ -n "$TRIGGER_ISSUE_NUMBER" ] && [
  ! -s "issue-comment.md" ] && [ ! -s "pr-comment.md" ]` is the right gate.
  `[ ! -s ]` covers both "file doesn't exist" and "file is zero bytes" —
  both are equivalent failure modes from the post-run perspective.
- `common.md:96-97`, `:107-108`, `:117-118`, and `:122-123` — four
  parallel "use `write_file`" tightenings in the PR-prep protocol. The
  diffs are surgical: same wording structure, just inserting "Use the
  `write_file` tool to" before the verb. This consistency matters because
  the brain reads these as a single procedural document and inconsistent
  phrasing produces inconsistent agent behaviour.
- `interactive.md:33-34`, `:58-60`, `:69-73` — same pattern in the
  interactive-mode branch, ending with the strongest directive (the
  all-caps `MUST` / `DO NOT`). The `MUST` clause includes the *causal
  explanation* ("The workflow relies on `issue-comment.md` being created in
  the workspace"), which is the right prompt-engineering shape: explain
  why, not just what.

## Reasoning

The triage here is interesting because the symptom (bot posting "invalid
response" to a comment) was a compound failure: when the agent's text-only
response was streamed to stdout instead of `write_file`'d to the expected
output file, the next run inherited `issue-comment.md` from the previous
artifact (because of the blanket restore), and posted *that* — which was
unrelated to the new trigger. So the user-visible bug was "bot replies to
my new comment with the answer to someone else's old comment."

The fix is correctly multi-layered:

- **Prevention**: the brain prompts now explicitly require `write_file`,
  closing the most common path to the failure.
- **Containment**: the selective restore stops stale ephemeral state from
  contaminating fresh runs even if the agent fails again.
- **Detection/UX**: the fallback message ensures the user gets *some*
  response (with a link to the actions log) rather than silence-or-stale.

Each layer addresses a distinct failure mode, and removing any one of them
leaves a real regression vector. The selective restore in particular is the
right shape — keeping `lessons-learned.md` and `history/*.csv` (the genuine
cross-run state) while explicitly *not* restoring transient comment files
maps the inclusion list to the actual persistence-requiring artifacts.

The all-caps `MUST` / `DO NOT` directive in `interactive.md:69-73` reads as
shouting but is appropriate for prompt engineering: LLM compliance with
imperatives is empirically higher when explicit emphasis is paired with a
causal "why" clause, and silently dropping back to console-output is
exactly the failure mode this PR exists to fix.

## Nits (non-blocking)

1. `gemini-cli-bot-brain.yml:88` — the temp-dir is `.temp_brain_data`
   (leading dot, hidden in `ls`), which is fine but inconsistent with the
   `tools/gemini-cli-bot/` non-hidden convention. Visible name like
   `_brain_restore_tmp/` would be slightly easier to debug if a step
   fails mid-restore. Minor.
2. The `cp ...` calls swallow all errors via `2>/dev/null || true`. If the
   artifact restored but the *contents* changed shape (e.g. lessons file
   moved to a subdirectory in a future bot revision), this would silently
   degrade to "no lessons restored" without surfacing in the run log. A
   `[ -f .temp_brain_data/.../lessons-learned.md ] || echo "lessons file
   not in artifact"` check before the cp would help.
3. The fallback message at `:147` hardcodes "Gemini CLI Bot" — if this
   workflow gets renamed or forked, the user-facing text will go stale.
   Templating it from a workflow-level env var would be more robust.
4. The brain-prompt edits don't add a counter-example (a "do not do this:
   `print(response)`" block). A negative example often improves LLM
   compliance more than positive instructions alone.
5. `pr-comment.md` is mentioned in the fallback gate at `:144` but the
   fallback only writes `issue-comment.md`. If the trigger was on a PR
   comment specifically, the fallback message lands in the wrong file
   (or at least, in a file the workflow's downstream-poster doesn't read
   from for PRs). Worth confirming `issue-comment.md` is the canonical
   fallback channel for both surfaces.
