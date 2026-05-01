# PR #25305 — fix(prompt): remove shell mode suspension from input traits

- Repo: sst/opencode
- Author: adavila0703
- Head: `dfbda4f2a8fbffd762066b805ac25bc179d0d127`
- URL: https://github.com/sst/opencode/pull/25305
- Verdict: **needs-discussion**

## What lands

One-line change at `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:568`:

```diff
-      suspend: !!props.disabled || store.mode === "shell",
+      suspend: !!props.disabled,
```

Diff stops gating input on `store.mode === "shell"`. The accompanying line
569 still keeps `status: store.mode === "shell" ? "SHELL" : undefined` so
the visual indicator is preserved.

## Specific findings

- The `suspend` flag on `input.traits` is what tells the TUI input layer to
  ignore key events. Originally, when the user toggled into shell mode, the
  prompt input was *suspended* — i.e. captured-but-ignored — so keys went
  to the shell backend instead of being inserted into the TUI prompt. With
  this PR, `suspend` is now only true when `props.disabled` is set, which
  means in shell mode the prompt input will actively process keystrokes
  again.
- The PR description / body context isn't included in the diff but the
  `status: store.mode === "shell" ? "SHELL" : undefined` line is left in
  place at `:569`, suggesting the author intends shell mode to *visually
  remain shell mode* while letting the prompt accept text. This is a
  coherent UX choice (e.g. so users can type a command while a previous
  shell invocation is still running) but it directly inverts the prior
  suspension contract.
- Question for maintainers: what happens to the shell command stream now?
  Pre-PR, in shell mode keystrokes were routed to the shell backend
  (because the prompt was suspended and didn't claim them). Post-PR, the
  prompt claims them and they go into `input.value`. The shell backend
  presumably still wants its stdin — does this PR also re-route the shell
  stdin to read from `input.value`, or does it just silently break shell
  input forwarding?
- No accompanying test was added. For a one-line behavior inversion in a
  high-traffic interactive surface, that's the wrong test floor.

## Risks

- If shell mode previously relied on `suspend: true` to forward keys to the
  shell process, this PR will silently break shell-typed-input. Users
  toggling into shell mode would still see the SHELL indicator but their
  keystrokes would be captured by the prompt and never reach the shell.
- If shell mode is now intended to be "type a command in the prompt, then
  press Enter to send to shell", that's a meaningful UX change that
  deserves docs and a test, not a one-line diff with no body explanation
  in the diff stream.

## Verdict

**needs-discussion**. The diff is mechanically tiny but inverts a
load-bearing contract on the prompt's input pathway. Need the author or
maintainer to confirm: (1) what the new shell-mode keystroke routing is,
(2) whether the rest of the shell-input pipeline already reads from
`input.value` or expects keys via the suspend-route, (3) why no test was
added for a behavior inversion.
