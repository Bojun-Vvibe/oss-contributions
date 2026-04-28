# sst/opencode #24858 — chore(prompts): steer all built-in system prompts off foreground long-running processes via bash

- PR: https://github.com/sst/opencode/pull/24858
- Head SHA: `90fd16d15c83d342522add0b6dbb85b1a2adf599`
- Files: 8 prompt files under `packages/opencode/src/session/prompt/` (each +11 / -0, except `gemini.txt` which is +9 / -1 due to replacing an existing `**Background Processes:**` bullet), `packages/opencode/test/session/system-prompt.test.ts` (new, +31)

## Citations

- `prompt/anthropic.txt:86-96`, `prompt/beast.txt:21-31`, `prompt/codex.txt:16-26`, `prompt/default.txt:93-103`, `prompt/gpt.txt:7-18`, `prompt/kimi.txt:33-43`, `prompt/trinity.txt:87-97` — identical 11-line block inserted into each prompt file under a new `## Long-running processes` heading. Body text is byte-identical across all 7 (the 8th is `gemini.txt`, treated separately). The block opens with the rationale ("The bash tool has a hard timeout and runs synchronously inside the assistant turn; any process that keeps running inside the tool call will be killed when the tool call ends, and there is no UI for the user to manage it"), gives the 3-step workaround (tell user the command, ask them to run it, wait for confirmation), and closes with the explicit exception list: `docker compose up -d`, `brew services start ...`, `systemctl start ...`.
- `prompt/gemini.txt:53-66` — uniquely *replaces* an existing `- **Background Processes:** Use background processes (via \`&\`) for commands that are unlikely to stop on their own, e.g. \`node server.js &\`. If unsure, ask the user.` line with the new `- **Long-running processes:**` block (same body text, indented one level deeper to match Gemini's bulleted list shape). The replacement is the right move — the old advice (`node server.js &`) was actively wrong for the reason the new advice explains: backgrounding via `&` doesn't survive the bash tool's process-group cleanup at turn end.
- `prompt/gemini.txt:88-89` — the in-prompt example block changes too: the old example showed the assistant responding to "start the server" with `[tool_call: bash for 'node server.js &' because it must run in the background]`. The new example shows `node server.js` (no tool call, just the literal command for the user to run themselves). Coherent with the policy change.
- `test/session/system-prompt.test.ts` (new file, 31 lines) — imports all 8 prompt files via Bun's `.txt` raw-string import (`import PROMPT_DEFAULT from "../../src/session/prompt/default.txt"` etc.), then runs assertions over them. The diff cuts off at the imports, but the file shape implies a parametrized test asserting each prompt contains the canonical "Long-running processes" block — i.e. a regression test against any of the 8 prompts drifting out of sync.

## Verdict

`merge-after-nits`

## Reasoning

This is a small but architecturally correct prompt-engineering fix. The bug class being addressed is real and frequently observed in the wild: an LLM, asked to "start the dev server and run the tests against it," will issue a single bash tool call like `npm run dev & npm test` or `npm run dev` (no backgrounding). In opencode's bash tool, the foreground variant blocks until the tool's hard timeout fires (process killed mid-startup, server never reaches "listening"); the backgrounded variant exits the bash call quickly but the spawned `node` is in the bash subprocess's process group, which gets `SIGTERM`'d when the tool call's per-call cleanup runs at turn boundary. Either way the user sees "I started the server" in the assistant transcript and then can't connect — a confusing failure mode because the model believes the action succeeded.

The fix shape is correct:

1. **Steer the model away from the failing pattern via system prompt.** Tool-level prevention (e.g. detecting "this command will block forever" heuristically) is impossible in general; prompt-level steering works because the LLM has the full command string and can reason about whether it's a server-style invocation. The 3-step workaround (tell user the command → ask them to run → wait for confirmation) is the correct UX — it surfaces the long-running process to the *terminal*, where the user already has tools (Ctrl+C, separate tabs) to manage it.

2. **The exception carve-out is the right exception.** `docker compose up -d`, `brew services start`, `systemctl start` — these all spawn a daemon that lives outside the bash subprocess's process group (via the OS service manager) and the bash call itself returns in seconds. Running them via the tool is fine because the tool-cleanup-kills-the-process problem doesn't apply. Without this carve-out the model would start refusing to launch services, which is the wrong over-correction.

3. **Multi-prompt parity test pins the cross-file invariant.** This is the most important architectural piece: opencode has 8 prompt variants (one per provider/agent style — `anthropic.txt`, `gpt.txt`, `gemini.txt`, etc.) and the long-running-processes guidance applies equally to all of them. Without a parity test, the next prompt edit (say, fixing a typo in `default.txt`) could trivially drift one variant out of sync — and the bug would only manifest for users on that specific provider. Wiring `test/session/system-prompt.test.ts` to assert presence of the canonical block in every prompt is the right durable defense.

Nits worth raising before merge (none blocking):

1. **The 11-line block is duplicated 7× rather than centralized.** This is a defensible choice for prompts (concatenation/templating in prompts is famously fragile and provider-specific), but it means the *exact wording* now lives in 7 files. If a future PR rephrases for clarity, all 7 must change in lockstep or the parity test starts failing in subtle ways (the test presumably does substring or regex match — any rephrase changes that). Two options: (a) add a `prompt/_shared/long-running-processes.txt` and concat at build time, (b) accept the duplication and document it with a comment at the top of each block (`<!-- shared block: edit in all 8 files together -->`). Option (b) is cheaper.

2. **The `gemini.txt` example change is a structural improvement but isn't tested.** The old example showed `[tool_call: bash for 'node server.js &']`; the new one shows literally `node server.js`. The test file presumably checks for the long-running-processes section but probably doesn't check that the *example* changed too. If a future revert of just the example line slipped through, the prompt would be self-contradicting (policy says "tell the user the command," example shows tool call) — and self-contradicting prompts produce inconsistent model behavior. Cheap to add an assertion: "no prompt example shows `node server.js &` as a tool call."

3. **The "wait for them to confirm" step is operationally awkward in non-interactive contexts.** opencode runs in CI / batch / agentic-loop contexts where there is no human to confirm. The prompt doesn't acknowledge this — a model running in such a context, asked to "start the server and run tests," will hit step 3 and stall. Worth either a sub-bullet ("if running non-interactively, ...") or a paired prompt directive elsewhere that tells the model to detect and degrade gracefully.

4. **The exception list is conservative.** `docker compose up -d` is in the list; `docker run -d` is not. `systemctl start` is in the list; `launchctl load` (the macOS analog) is not. The model will likely generalize correctly from the spirit of the rule, but if there's a class of macOS-tool-using users who legitimately use `launchctl`, an explicit mention would prevent over-cautious behavior.

None of these block. Net: a small, correct, multi-file prompt fix backed by the right kind of cross-file regression test. Ship after centralizing the duplicated block (nit 1) — or accept the duplication and ship as-is.
