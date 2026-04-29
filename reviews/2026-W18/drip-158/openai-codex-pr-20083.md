# openai/codex#20083 — Mark goals feature as experimental

- PR: https://github.com/openai/codex/pull/20083
- Head SHA: `6430e1d8fe161d0a7a24bf047f65bd74ddaf569d`
- Author: etraut-openai
- Diff: +5/-1 across `codex-rs/features/src/lib.rs`

## What changed

Single-file stage transition for the `Goals` feature in the `FEATURES` registry at `codex-rs/features/src/lib.rs:949-957`. The entry's `stage` field flips from `Stage::UnderDevelopment` to the structured `Stage::Experimental { name: "Goals", menu_description: "Set a persistent goal Codex can continue over time", announcement: "" }`. `default_enabled: false` is preserved, so the feature stays opt-in.

## Observations

This is the canonical promotion shape used by every other experimental feature in this file — `name` becomes the surface label, `menu_description` is the in-product feature-toggle blurb, and `announcement: ""` is the no-banner sentinel that other entries (e.g. recent ones in this same `FEATURES` slice) also use when there's no callout-worthy launch. Because `default_enabled` stays `false`, no end-user is opted in by this commit; the only behavioral delta is that the feature now appears in the `Experimental` panel rather than being hidden under `UnderDevelopment`, where users with the dev flag could already see it but other users could not.

The empty `announcement` is worth a glance — across the file most experimental entries either have a meaningful string or `""`. If product wants this surfaced in the next release-notes pass, the empty literal will silently swallow that. Not blocking; the convention is established.

The `menu_description` reads cleanly: imperative verb ("Set") plus the load-bearing noun ("persistent goal") plus the differentiating clause ("can continue over time"). It matches the tone of neighbors in `FEATURES`. No banned content.

There's no test change because the registry shape is enforced by the `Stage` enum's exhaustive match downstream; any consumer that switches on `Stage` would have to handle the new variant by construction.

## Verdict

`merge-as-is`
