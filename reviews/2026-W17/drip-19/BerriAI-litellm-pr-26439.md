---
pr_number: 26439
repo: BerriAI/litellm
head_sha: 79517bc6282c43d0d844b6d3f7b2597e3c2735bd
verdict: merge-after-nits
date: 2026-04-25
---

# BerriAI/litellm#26439 — pass `output_config` through to backends that accept it (vertex+anthropic)

**What changed.** 7 files / +526 / −82. Three landing zones: (a) `litellm/llms/anthropic/experimental_pass_through/adapters/handler.py` adds `ANTHROPIC_ONLY_REQUEST_KEYS = frozenset({"output_config"})` (line 30) and merges it into `excluded_keys` (line 240ish) so non-Anthropic backends don't get the raw Anthropic-shaped key after the translator already mapped it. (b) `adapters/transformation.py` `_translate_output_format_to_openai` (line 1081) now ALSO consumes `output_config.format`, not only the legacy top-level `output_format` field. (c) `vertex_ai_partner_models/anthropic/output_params_utils.py` (new, 50 lines) exports `sanitize_vertex_anthropic_output_params` and `VERTEX_UNSUPPORTED_OUTPUT_CONFIG_KEYS = frozenset({"effort"})`, called from both `experimental_pass_through/transformation.py` and the chat-completion `transformation.py`. The previous code blindly `pop`'d both `output_format` and `output_config`, dropping structured-output schemas entirely.

**Why it matters.** Closes #23380 (output_config dropped) and supersedes 4 stalled community PRs (#23475, #23396, #23706, #22727) that each fixed one slice. Real user impact: structured outputs and adaptive-thinking effort were silently no-ops on Vertex Claude.

**Concerns.**
1. **`ANTHROPIC_ONLY_REQUEST_KEYS` defined as `frozenset[str]`** (line 36) using PEP 585 generic-frozenset syntax. Works on 3.9+. The repo's `setup.py` should already require ≥3.9 (most of the 3.x codebase assumes it), but worth a quick grep — if any test target still runs 3.8, this fails at import.
2. **The `extra_kwargs is not None` change vs `extra_kwargs or {}`** (line 215) is correct and the comment explains why (preserving caller-passed empty dict for fallback inference path). Good. But the same module re-asserts `extra_kwargs = extra_kwargs or {}` at line ~242 in the older branch — diff replaces with `# NOTE: extra_kwargs was already coerced ...`. Fine; just verify no execution path enters the second block without going through the first.
3. **`sanitize_vertex_anthropic_output_params` strips entire `output_config` if the only key was `effort`** (line 47 of new file). Correct and tested. But `output_format` is forwarded as-is per the docstring ("Vertex AI Claude accepts it"). Worth a confirming integration test against current Vertex API surface — Anthropic's own Vertex docs page has historically lagged the "accepts X" list. If Vertex returns 400 on `output_format` today (regardless of payload validity), this won't actually fix the consolidated bug.
4. **`isinstance(output_config, dict)` non-dict path drops silently** (line 39). For an upstream caller passing a Pydantic model accidentally, this swallows the data. Either log a warning or `raise TypeError`. "Drop and continue" leads to the same silent-loss complaint the PR is trying to fix.
5. **CodeQL cyclic-import note** in the new file's docstring is a nice piece of breadcrumb documentation. Keep.
6. **Co-Authored-By trailers** for the four superseded PR authors are present per the body — good open-source citizenship; verify they're in the actual commit metadata (not visible in diff alone).

Substantively the right fix and the consolidation is well-justified. Land after a real Vertex-side integration probe (#3) and the silent-drop hardening (#4).
