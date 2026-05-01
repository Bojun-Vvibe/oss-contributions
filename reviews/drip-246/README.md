# drip-246 (2026-05-02)

8 fresh PR reviews across 6 OSS AI-coding-agent repos. Local artifacts
only; nothing was submitted upstream. Verdict mix:

- merge-as-is: 2 (sst/opencode#25309, openai/codex#20628)
- merge-after-nits: 3 (openai/codex#20630, BerriAI/litellm#26988,
  block/goose#8950)
- request-changes: 1 (google-gemini/gemini-cli#26337)
- needs-discussion: 2 (sst/opencode#25305, QwenLM/qwen-code#3781)

## Themes

- **Idempotence + source-aware skips break feedback loops** (opencode
  #25309) — three independent gates (no-op write guard, mouse-source
  scroll skip, stale-prop abort) at the source of a hover↔scroll
  ping-pong are the right minimum surface.
- **Capability probes should be exhaustive at the gate, not at the
  consumer** (codex #20628) — `SystemBwrapCapabilities { argv0, perms }`
  destructured with `supports_perms: true` in the pattern is the most
  concise way to express "must succeed AND must have perms" and means
  the launcher consumer never has to re-check.
- **Diagnostic suppression is the inverse of robustness** (gemini-cli
  #26337) — `2>/dev/null` on a CI version-check command discards the
  exact stderr that the on-call needs when `$gemini_version` comes back
  empty. Robust-by-name, fragile-by-effect.
- **Privilege-escalation via fallback-on-error** (litellm #26988) —
  catching DB-connection errors during *secondary* common_checks and
  issuing an unrestricted fallback token silently erases the *primary*
  token's budget/team/model restrictions. Fix: split denial exceptions
  (re-raise) from DB exceptions (preserve original token, log at
  warning) from coding-bug exceptions (route through the existing
  handler).
- **Bracket-marker protocols need allowlist + size cap before they
  ship** (qwen-code #3781) — `[IMAGE: /path]` parsed out of model output
  and fed into a CDN upload with no path or size validation is an
  arbitrary-file-exfiltration vector activated by prompt injection.
  Same shape as the path-traversal class of bug; the gate belongs at
  the marker-parser, not at the upload site.
- **Per-message capability reminders pollute the conversation** (qwen-
  code #3781) — prepending an instruction string to every inbound user
  message costs tokens, surfaces as if the user said it, and is a
  workaround for the model forgetting which is better solved by system-
  prompt re-emission or tool-call elevation.
- **One-line behavior inversions need test floors** (opencode #25305) —
  removing `|| store.mode === "shell"` from `input.traits.suspend`
  flips a load-bearing contract on the prompt's input pathway. No
  test, no body explanation in the diff stream, and the visible
  `status: "SHELL"` indicator is preserved while the underlying suspend
  contract changes — exactly the recipe for a regression that ships.
- **Settings-modal decomposition is a compound refactor** (goose #8950)
  — three concerns bundled (per-section componentization, document-
  level keydown, removed inter-section transition) where the title
  only captures one. The transition removal is a UX change that
  deserves either explicit callout or restoration.

## Repos covered

- sst/opencode (×2: #25309, #25305)
- openai/codex (×2: #20630, #20628)
- BerriAI/litellm (×1: #26988)
- google-gemini/gemini-cli (×1: #26337)
- block/goose (×1: #8950)
- QwenLM/qwen-code (×1: #3781)

6 repos, no overlap with drip-245.
