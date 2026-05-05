# block/goose #9035 — fix(openai): accept null tool_call arguments in streaming chunks

- Head SHA: `d563dfbb39ebc375c7f1674e3cf4bb3101a6351c`
- Diff: +23 / -1 across 1 file (`crates/goose/src/providers/formats/openai.rs`)

## Findings

1. Correct root-cause diagnosis. `DeltaToolCallFunction.arguments` is a `String` typed field with `#[serde(default)]` (visible at `crates/goose/src/providers/formats/openai.rs:46-48` pre-fix). `serde(default)` only fires for the *missing-key* case, so a streaming chunk that emits the explicit JSON `"arguments": null` (per the linked issue #8991 from deepseek-v4-pro) blew up the entire chunk parse with `invalid type: null, expected a string`, causing the whole tool call to be silently dropped from the streaming accumulator.
2. Fix is the minimal, targeted approach: a custom `deserialize_null_default_string` helper at `:35-40` that delegates to `Option::<String>::deserialize` and unwraps to `String::default()` (empty string) on `None`. The `#[serde(default, deserialize_with = "...")]` attribute combo at `:48` makes the helper handle both the missing-key and explicit-null cases uniformly. The field stays as `String` (not `Option<String>`), so the downstream `push_str` / clone sites in the streaming accumulator that concatenate per-chunk delta arguments don't need to change — the explicit author note "Kept it as `String` rather than switching to `Option<String>` so the change is local to this one field" is exactly the right call to minimize blast radius.
3. Test at `:2447-2459` covers the three relevant cases: explicit `null` (`r#"{"arguments":null}"#` → `""`), missing key (`r#"{}"#` → `""`), and populated (`r#"{"arguments":"{\"k\":1}"}"#` → preserved). Good coverage for a 4-line semantic change.
4. Concern (minor): the empty-string-from-null mapping is semantically lossy — a downstream consumer that sees `arguments == ""` cannot distinguish "the model sent an empty string" from "the model sent null" from "the field was missing". For the streaming-accumulator use case this is fine (all three should accumulate to nothing for this chunk), but if any future consumer wants to e.g. skip whole tool calls where `arguments` was explicitly null, this fix forecloses that. A one-line comment at `:48` documenting the deliberate collapse would prevent future debugging confusion.

## Verdict

**Verdict:** merge-as-is
