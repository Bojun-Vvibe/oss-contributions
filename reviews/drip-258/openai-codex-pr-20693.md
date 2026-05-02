# openai/codex PR #20693 — Preserve image detail in app-server inputs

- PR: https://github.com/openai/codex/pull/20693
- Head SHA: `5fad2945b670c00faf5b13f055d7a0269890711e`
- Author: @fjord-oai (Curtis 'Fjord' Hawthorne)

## Summary

Adds an optional `detail: Option<ImageDetail>` field to the `Image` and `LocalImage` variants of app-server `UserInput` (and the v2 schema), preserves it across v2↔core conversions, and applies the requested detail when converting to Responses-API content items. `ImageDetail` is `auto | low | high | original`, with `original` getting special handling for local images. Schemas are regenerated; affected callers and tests get the new field defaulted to `None`.

## Specific references from the diff

- `codex-rs/app-server-protocol/schema/json/ServerNotification.json:1930-1939` — new `ImageDetail` enum with values `["auto","low","high","original"]`.
- `codex-rs/app-server-protocol/schema/json/ClientRequest.json:4358-4395` and `ServerNotification.json:4414-4451` — adds `detail` (Optional, default `null`) to both `image` (URL) and local-image (`path`) variants.
- `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.schemas.json:17836-17878` and `codex_app_server_protocol.v2.schemas.json:15722-15764` — same additions in the bundled top-level and v2 schemas.
- `codex-rs/analytics/src/analytics_client_tests.rs:229-232` and `:346-349` — call sites updated: `UserInput::Image { url, detail: None }` and `UserInput::LocalImage { path, detail: None }`.
- Author validation: `cargo fmt`, `cargo run -p codex-app-server-protocol --bin write_schema_fixtures`, plus three targeted `cargo test`s (`image_user_input_preserves_requested_detail`, `schema_fixtures_match_generated`, `user_input_into_core_preserves_image_detail`).

## Verdict: `merge-after-nits`

Pure additive protocol change with `default: None`, tests covering both the protocol round-trip and the v2-into-core conversion, and schema fixtures regenerated. Backwards-compatible. Two small things keep it off `merge-as-is`.

## Nits / concerns

1. **`original` semantics need to be documented in the enum.** The PR body says "`original` handling for local images" but the schema just lists the four enum values. Add a Rust doc comment on the `ImageDetail` enum (and have `schemars` carry it into the JSON `description`) explaining: (a) `original` is only meaningful for local images, (b) what happens if a remote `Image { detail: Original }` comes in — does the conversion downgrade to `auto`, error, or pass through? Today, a client sending `{"type":"image","url":"...","detail":"original"}` has undefined behavior from the schema alone.
2. **Default behavior parity.** Confirm in the PR that for clients that don't set `detail` (the `default: null` case), the wire bytes sent to Responses are byte-identical to today. The diff modifies the conversion but the test name `image_user_input_preserves_requested_detail` doesn't assert "absent ≡ pre-PR". A second assertion `detail = None ⇒ no `detail` key in the outgoing Responses content item` would lock in non-regression for existing callers.
