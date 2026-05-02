---
repo: charmbracelet/crush
pr: 2755
head_sha: 35541f22ed6a097729fe0d7e9619d35331029cf1
title: "fix: fix thinking on/off toggle for certain openai-compat providers"
verdict: merge-as-is
reviewed_at: 2026-05-03
---

# Review: charmbracelet/crush#2755 — `fix: fix thinking on/off toggle for certain openai-compat providers`

**Head SHA:** `35541f22ed6a097729fe0d7e9619d35331029cf1`
**Stat:** +36 / −21. State: MERGED. Author: andreynering.
Companion change: charmbracelet/fantasy#220.

Reviewing post-merge for the drip log; comments are advisory.

## What it changes

Fixes the "thinking" (reasoning) toggle path for openai-compat
providers that need provider-specific request shapes. Three files:

1. **`go.mod` / `go.sum`**: bumps `charm.land/catwalk v0.38.0 →
   v0.39.1` and `charm.land/fantasy v0.22.0 → v0.23.0`. The companion
   `charm.land/fantasy#220` is the upstream change these versions
   pull in.
2. **`internal/agent/coordinator.go`**: deletes the special-cased
   provider-type switching block at the old lines ~284–293 that
   re-mapped `"hyper"` provider type to `anthropic.Name`,
   `openai.Name`, `google.Name`, or `openaicompat.Name` based on
   substring match against `model.CatwalkCfg.ID` (`"claude"`, `"gpt"`,
   `"gemini"`). That was the wrong abstraction — it forced one
   provider config to masquerade as multiple Fantasy providers.

   Replaces it by collapsing `hyper.Name` into the
   `openaicompat.Name` switch arm (line 364:
   `case openaicompat.Name, hyper.Name:`) and adding an explicit
   per-provider `extra_body` shape inside that arm:

   ```go
   switch providerCfg.ID {
   case hyper.Name:
       extraBody["thinking"] = model.ModelCfg.Think
   case string(catwalk.InferenceProviderIoNet):
       extraBody["chat_template_kwargs"] = map[string]any{
           "thinking": model.ModelCfg.Think,
       }
   case string(catwalk.InferenceProviderZAI):
       if model.ModelCfg.Think {
           extraBody["thinking"] = map[string]any{"type": "enabled"}
       } else {
           extraBody["thinking"] = map[string]any{"type": "disabled"}
       }
   }
   mergedOptions["extra_body"] = extraBody
   ```

## Assessment

- Removing the substring-based provider rewrite is the right call. The
  old code was guessing `claude` ⟹ Anthropic shape, `gpt` ⟹ OpenAI
  shape from model IDs — that's a maintenance trap (any future model
  whose ID happens to contain those substrings would be silently
  rewritten). The new code keeps the user's declared `providerCfg.Type`
  and only diverges *request body shape*, which is the actual axis of
  variation.
- The three concrete provider IDs handled (`hyper`, IoNet, ZAI) each
  use a different "thinking" wire shape — boolean, nested under
  `chat_template_kwargs`, and `{type: enabled|disabled}` respectively.
  This matches the messy reality of openai-compatible providers each
  inventing their own reasoning-toggle convention.
- The two `TODO` comments inside the switch (`// TODO: Abstract this
  in Fantasy somehow?` and `// TODO: Allow custom providers to
  specify how to set this?`) are honest — the right long-term fix is
  in Fantasy, and that's already in flight (companion fantasy#220).
- `extraBody := make(map[string]any)` is created unconditionally and
  always assigned to `mergedOptions["extra_body"]`, so providers not
  in the inner switch get an empty `extra_body` written to options.
  That's fine for the openai-compat path (downstream handles missing/
  empty `extra_body`), but worth noting for any future reader.
- `model.ModelCfg.Think` is a bool, so for `hyper` this writes
  `extraBody["thinking"] = false` even when thinking is off, rather
  than omitting the key. Match the upstream provider's expected
  contract — if the provider distinguishes `thinking: false` from
  "key absent", this matters. The fact that this PR was merged
  suggests it doesn't.
- The dependency bumps catwalk → 0.39.1 and fantasy → 0.23.0 are
  necessary because the new switch references `hyper.Name` and
  `catwalk.InferenceProviderIoNet` / `catwalk.InferenceProviderZAI`
  symbols that come from the bumped versions.

## Nits

(Post-merge — advisory only.)

- (Non-blocking) The two `TODO` comments would benefit from linking
  the fantasy issue/PR they're tracked by, so a future reader knows
  whether they're "still open" or "fixed upstream, just need to
  consume". Could be a 1-line follow-up.
- (Non-blocking) The unconditional `mergedOptions["extra_body"] =
  extraBody` (even when `extraBody` is empty) might surprise downstream
  Fantasy code that distinguishes "no extra body" from "empty extra
  body". Worth a glance but probably no-op.

## Verdict

**`merge-as-is`** (already merged) — correct removal of a guessy
substring-based provider rewrite, replaced with explicit per-provider
request shaping. The TODOs honestly admit Fantasy is the right home
for this dispatch.
