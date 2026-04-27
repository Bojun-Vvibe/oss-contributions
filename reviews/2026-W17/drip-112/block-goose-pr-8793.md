# block/goose PR #8793 — fix(goose): exclude preprompt from session title generation

- **PR**: https://github.com/block/goose/pull/8793
- **Author**: @matt2e
- **Head SHA**: `29cde95336c407ea27032475766554d64099eb3e`
- **Files**: `crates/goose/src/providers/base.rs`, `crates/goose/src/providers/cli_common.rs`

## Summary

Session-title generation was blending preprompt content (assistant-audience-only blocks attached to user messages) into the user-message text passed to the title-generation prompt, causing titles like "Following workspace conventions" instead of "Refactor auth middleware". The fix: (1) `get_initial_user_messages` now uses `c.filter_for_audience(rmcp::model::Role::User)` to extract only blocks visible to the user, dropping preprompt content; (2) a new `get_preprompt_context` companion extracts the *complementary* set (assistant-only blocks) and passes it as a clearly-fenced `---BEGIN BACKGROUND CONTEXT---` section in the title-generation prompt with explicit instructions not to base the title on it; (3) the simple-description path strips the new fence before extracting the user content.

## Verdict: `merge-after-nits`

Solid fix that addresses both halves of the problem: don't *contaminate* the title with preprompt text, but also don't *hide* the preprompt from the model entirely (it's still useful as background context for disambiguation). The marker-based stripping in `cli_common.rs` is the right place to reverse the addition.

## Specific references

- `crates/goose/src/providers/base.rs:740-751` — the new body of `get_initial_user_messages`:
  ```rust
  .map(|m| {
      m.content
          .iter()
          .filter_map(|c| c.filter_for_audience(rmcp::model::Role::User))
          .filter_map(|c| c.as_text().map(|s| s.to_string()))
          .collect::<Vec<_>>()
          .join("\n")
  })
  ```
  Replaces the previous `m.as_concat_text()` which concatenated all content blocks regardless of audience. The new chain is `filter_for_audience(User)` → `as_text()` → `join("\n")`, which correctly drops both (a) blocks audience-restricted away from the user and (b) non-text blocks (images, resources). The doc comment update at line 738 (`"...filtering out assistant-only content (e.g. preprompt blocks)"`) is precise.
- `crates/goose/src/providers/base.rs:756-773` — the new `get_preprompt_context` is the *complement* of the above: it iterates the first user message's content, keeps only blocks where `c.filter_for_audience(rmcp::model::Role::User).is_none()` (i.e. blocks NOT visible to the user), pulls their text, and joins. The `take(1)` is intentional — only the first user message typically carries preprompt; later messages are interactive turns. This is a defensible heuristic but worth a doc comment explaining why `take(1)` and not `take(MSG_COUNT_FOR_SESSION_NAME_GENERATION)`.
- `crates/goose/src/providers/base.rs:790-798` — the prompt formatting:
  ```rust
  let preprompt_section = if preprompt_context.is_empty() {
      String::new()
  } else {
      format!(
          "---BEGIN BACKGROUND CONTEXT (for understanding only, do NOT base the title on this)---\n{}\n---END BACKGROUND CONTEXT---\n\n",
          preprompt_context
      )
  };
  ```
  The "do NOT base the title on this" instruction inside the fence is good — models generally respect explicit role-segregation when the segregation is visually obvious. The empty-string short-circuit when no preprompt exists keeps the prompt clean for the common case.
- `crates/goose/src/providers/base.rs:799-803` — the format string changes from `"{}\n{}\n{}\n\n{}"` (4 args) to `"{}{}\n{}\n{}\n\n{}"` (5 args, with `preprompt_section` first). Note there's no `\n` between `preprompt_section` and `SESSION_NAME_BEGIN_MARKER` because `preprompt_section` already ends with `\n\n` when non-empty. Defensible but easy to miss in review.
- `crates/goose/src/providers/cli_common.rs:58-62` — the corresponding strip in `generate_simple_session_description`:
  ```rust
  let text = text
      .rfind(SESSION_NAME_BEGIN_MARKER)
      .and_then(|idx| text.get(idx..))
      .unwrap_or(text);
  ```
  Uses `rfind` (not `find`) — important because if the title model echoed the marker into its own response, the *last* occurrence is the boundary we want. Good choice. The `.and_then(|idx| text.get(idx..)).unwrap_or(text)` chain falls back to the original text if `rfind` returned `Some` but the slice is somehow invalid (defensive).

## Nits

1. `get_preprompt_context`'s `take(1)` deserves a comment. Why only the first user message? Is preprompt definitionally first-message-only? If so, document. If not, this might miss preprompt attached to a follow-up message (rare but possible).
2. The format string change at `base.rs:799-803` from `"{}\n{}\n{}\n\n{}"` → `"{}{}\n{}\n{}\n\n{}"` relies on `preprompt_section` self-terminating with `\n\n`. A future refactor that changes the format helper might miss this invariant. Consider explicitly always including the trailing newline in the format string and conditionally producing `preprompt_section` without trailing newline — slightly more uniform.
3. No new tests in the diff (visible). A regression test would construct a `Conversation` with a user message containing both a User-audience text block and an Assistant-audience text block, call `generate_session_name`, and assert the prompt sent to the model contains the User text inside `SESSION_NAME_BEGIN_MARKER` markers and the Assistant text inside the `BACKGROUND CONTEXT` fence — and crucially, that the User text does NOT contain the Assistant text. Both `get_initial_user_messages` and `get_preprompt_context` should also have direct unit tests on a synthetic conversation.
4. The string `"---BEGIN BACKGROUND CONTEXT (for understanding only, do NOT base the title on this)---"` is a one-shot marker; consider centralizing it in the same `cli_common` module as `SESSION_NAME_BEGIN_MARKER` so future code that strips it (similar to the `rfind` in `cli_common.rs:60`) doesn't have to duplicate the literal.
5. `c.filter_for_audience(rmcp::model::Role::User).is_none()` is the semantic check for "this is preprompt", but it conflates "audience excluded the User role" with "audience filter returned None for any other reason" (e.g. a malformed audience block). Probably fine in practice, but worth one-line comment in `get_preprompt_context` clarifying the assumption.

## What I learned

Audience-restricted content blocks are an underused tool for solving "this prompt-input contaminates that prompt-output" problems. The right pattern, demonstrated here, is to **partition** the conversation along the audience axis (User-visible vs Assistant-only) and use the partitions for *different downstream prompts*: User-visible content is what gets summarized / titled / displayed, Assistant-only content is what gets passed as background-context for disambiguation. The fenced `BACKGROUND CONTEXT` block with explicit "do not base the output on this" instruction is also a useful pattern — it gives the model the information it needs to disambiguate ("this session is about a refactor, not the workspace conventions") without letting the conventions text leak into the title. The `rfind` (vs `find`) for the strip is a small but real correctness win when the model echoes its own input.
