# sst/opencode #24772 — fix(acp): accept https:// URIs in image content blocks

- PR: https://github.com/sst/opencode/pull/24772
- Author: truenorth-lj
- Head SHA: `3cfbef4ed260`
- Diff: 1+/1- across 1 file (`packages/opencode/src/acp/agent.ts`)

## Verdict: merge-as-is

## Rationale

- **Surgical typo fix at `packages/opencode/src/acp/agent.ts:1394`.** The pre-fix code reads `part.uri.startsWith("http:")` — a literal 5-char prefix that matches `http://...` but rejects `https://...` (position 4 is `s`, not `:`). The fix expands to `(startsWith("http://") || startsWith("https://"))` which is the canonical pattern.
- **Codebase consistency argument is the load-bearing one.** PR body cites four other URL-prefix call sites that all already use the dual-prefix pattern: `session/instruction.ts:146,168`, `cli/cmd/import.ts:98`, `tool/webfetch.ts:23`, `config/config.ts:1270`. This was the only outlier — confirms typo origin, not deliberate http-only policy.
- **Failure mode is silent payload drop, not error.** When `part.uri = "https://..."` falls through the `case "image"` block, no `parts.push(...)` happens — the image content block is just gone from the prompt with no warning. This is the worst class of bug: cloud-storage signed URLs from GCS/S3 are *always* `https://`, so any ACP client doing two-stage uploads (the standard pattern for non-trivial images) loses the image silently. Fix promotes from "hidden data loss" to "works".

## Nits / follow-ups

- No test added. The author acknowledges this in the PR body and offers to add one if pointed at a pattern. A two-cell test (`http://` accepted, `https://` accepted) on the `case "image"` block in whatever the ACP-agent test file is would catch any future regression. Worth landing as a follow-up — the fix itself is too small and obvious to block on it.
- The deeper structural fix is to extract a shared `isHttpUrl(s)` helper used by all five call sites listed in the PR body — the next typo of this exact shape is one duplicate-prefix-check away. Out of scope for a typo PR but worth a follow-up issue.
- The case ordering at `:1394` — `data:` arm before `http:` arm — is fine as-is, but worth pinning with a comment that the order matters if either prefix overlaps (it doesn't here, but the next addition might).

## What I learned

`startsWith("http:")` instead of `startsWith("http://")` is a textbook off-by-one-character typo that's hard to spot in review because both look reasonable in isolation — the reader's eye fills in the missing `//`. The structural mitigation isn't "be more careful"; it's "ban the bare-prefix pattern" by introducing a typed `isHttpUrl()` helper. Anywhere five call sites do the same thing four different ways, the one different way is almost always the bug. The "silent drop" failure mode here is also a small-surface lesson: when a switch/if-chain has no `default` / `else` arm logging the unmatched case, every typo in a prefix check is a free silent-data-loss bug. A `verbose_logger.debug("unmatched part shape", part)` at the bottom of this `case "image"` block would have surfaced this within hours of the first ACP client trying a `https://` signed URL.
