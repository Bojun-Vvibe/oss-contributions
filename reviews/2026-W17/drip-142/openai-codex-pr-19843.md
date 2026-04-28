# openai/codex#19843 — `/plugins`: marketplace add + remove TUI flow (title misleading)

- **PR**: https://github.com/openai/codex/pull/19843
- **Author**: @canvrno-oai
- **Head SHA**: `b54c843`
- **Base**: `main`
- **State**: OPEN
- **Scope**: large — 17 files, +1078/-31. Title says "remove marketplace" but the diff *adds* both add and remove flows, with the bulk in `chatwidget/plugins.rs` (+569).

## Summary

Wires the `/plugins` slash-command TUI surface end-to-end with marketplace add/remove operations against the app-server protocol. Concretely:

1. New `AppEvent` variants: `OpenMarketplaceAddPrompt`, `MarketplaceAddLoaded`, `OpenMarketplaceRemoveLoading`, `FetchMarketplaceAdd`, `FetchMarketplaceRemove`, `MarketplaceRemoveLoaded`.
2. New `App` background-fetch helpers `fetch_marketplace_add` / `fetch_marketplace_remove` that send typed `MarketplaceAdd` / `MarketplaceRemove` requests over the app-server channel and dispatch the result back as an `AppEvent`.
3. New `ChatWidget` plumbing in `chatwidget/plugins.rs` (+569) that exposes a marketplace add prompt, a remove confirmation, and refreshes the on-disk plugin config when `cwd` matches the operation target.
4. New popups-and-settings test surface (`tests/popups_and_settings.rs` +232) with snapshot tests for the marketplace flows.
5. Drive-by error-message scrubs in `core-plugins/src/marketplace_add.rs` and `marketplace_add/source.rs` that drop the `source.display()` argument from user-facing errors and replace it with `"this source"` / `"local marketplace source path"`.

## Diff anchors

- **Title vs reality.** Title is `/plugins: remove marketplace`. The diff adds 1078 lines and removes 31; the work is dominated by *adding* the marketplace-add + marketplace-remove flows. The PR body is empty. This is a real readability problem for anyone scanning git log later — `git log --oneline | grep marketplace` will mislead. **Block on title fix.**
- `codex-rs/core-plugins/src/marketplace_add.rs:123-178` — three error messages drop `source.display()`. Trade-off: arguably an improvement for path-leak hygiene (don't echo full local FS paths in user-facing errors), but the resulting message ("...cannot be added from this source") is *strictly less debuggable*. Users hitting "marketplace 'X' is already added from a different source; remove it before adding this source" no longer see *which* source they tried. Recommend keeping the path in the error but routing it through a redaction helper instead — strip everything above `~`, or hash the absolute portion.
- `codex-rs/core-plugins/src/marketplace_add/source.rs:58-66` — `parse_marketplace_source` drops the offending `source` from the error. Same concern: "invalid marketplace source format; expected owner/repo, a git URL, or a local marketplace path" is a fine *type hint*, but `"invalid marketplace source format: {source}"` is what lets a user see the typo. Worse, the new message removes the actual bad input from logs/telemetry, making bug reports useless. Net: **this is a regression for support workflows**, dressed up as a polish change. Keep the `{source}` interpolation.
- `codex-rs/tui/src/app/background_requests.rs:90-138` — `fetch_marketplace_add` and `fetch_marketplace_remove` follow the existing `fetch_mcp_inventory` pattern (`tokio::spawn` → `request_typed` → `app_event_tx.send`). Right shape. Note that both helpers clone `cwd` and the source/name strings before the spawn so the `AppEvent` payload carries the original request context — required so the receiver can match the response to the originating cwd (see `chatwidget/plugins.rs` at the refresh-config-from-disk hook).
- `codex-rs/tui/src/app/event_dispatch.rs:200-260` — dispatches the new `AppEvent` variants. Each `OpenMarketplace*` arm is a thin forward to `chat_widget.*`. No new background-channel ordering invariants introduced.
- `codex-rs/tui/src/app_event.rs:282-335` — six new `AppEvent` variants. Naming is consistent with the existing pattern (`Open*Prompt`, `Fetch*`, `*Loaded`, `*Loading`). Worth confirming `MarketplaceRemoveLoaded` carries enough context (cwd + marketplace_name + display_name + result) for the receiver to render the toast without re-querying — it does.
- `codex-rs/tui/src/chatwidget/plugins.rs` (+569) — load-bearing. The new methods (`open_marketplace_add_prompt`, `submit_marketplace_add`, the marketplace_remove confirmation, the snapshot-driven popup rendering) are all gated behind explicit user action through the `/plugins` slash command. No silent network calls.
- `chatwidget/snapshots/codex_tui__chatwidget__tests__plugins_popup_*.snap` — three snapshot files updated/added. Worth a careful re-read of the diffs to make sure the snapshot contents don't accidentally hard-code an example local path that contains a username or workspace name.
- `tests/popups_and_settings.rs` (+232) — new integration tests. Verify coverage of the cwd-matching refresh path (the load-bearing piece — without it, adding a marketplace from inside the same repo doesn't show up until restart).

## What I'd push back on

1. **Fix the title.** "Add `/plugins` marketplace add+remove TUI flow" is what this is. The current title actively misleads.
2. **Empty PR body.** A 1078-line TUI change touching app-server protocol bindings deserves at least a 5-line "what / why / how to test." Reviewers shouldn't have to reverse-engineer this from the diff.
3. **Restore the offending `{source}` interpolation in error messages**, optionally behind a redaction helper. Removing it makes bug reports unactionable. Current state is "fail safely without saying why" — wrong trade-off for a developer tool.
4. **Snapshot review for path leakage.** Snapshot files in TUI tests have a long history of accidentally pinning developer-machine paths. Confirm the new `.snap` files use repo-relative paths only.
5. **No test for `OpenMarketplaceRemoveLoading` UX.** The variant exists; is anything actually rendering a spinner during the await? If the request takes >1s the user sees a frozen popup.

## Verdict

**request-changes** — block on (a) PR title rewrite, (b) restore the source argument in the three error messages (they're load-bearing for support), and (c) a non-empty PR description that names the protocol contract being added. The TUI plumbing itself looks correct, but I won't approve a 1078-line change with a misleading title and an empty body — those aren't optional.

Repo coverage: openai/codex (TUI app-event + chatwidget plumbing for marketplace add/remove flow against the app-server protocol).
