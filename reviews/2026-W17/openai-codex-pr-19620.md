# openai/codex #19620 — Escape turn metadata headers as ASCII JSON

- **Repo**: openai/codex
- **PR**: #19620
- **Author**: etraut-openai (Eric Traut)
- **Head SHA**: af96430a05c31f5a26e558b09306fb8aeac4150f
- **Base**: main
- **Size**: +109 / −4 across 6 files; bulk in new
  `codex-rs/utils/string/src/json.rs` (+85) and two new turn-metadata
  test cases that exercise non-ASCII workspace paths and metadata values.

## What it changes

Adds `to_ascii_json_string()` in `codex-rs/utils/string/src/json.rs:43-54`,
a `serde_json::Serializer` wrapped with a custom `Formatter` whose only
override is `write_string_fragment`. The formatter scans each fragment
for non-ASCII chars and emits them as `\uXXXX` escapes (UTF-16 code
units), passing ASCII bytes through verbatim. Two callers in
`core/src/turn_metadata.rs:72` and `:88` switch from
`serde_json::to_string` to the new helper so the resulting JSON survives
HTTP-header transport unchanged.

The two regression tests rename the temp repo from `repo` → `repo-東京`
(`turn_metadata_tests.rs:16`) and add an `origin: "東京"` client-metadata
entry (`:145`), then assert `header.is_ascii()` and that the original
unicode is preserved after a `serde_json::from_str` roundtrip.

## Strengths

- The core invariant — "JSON output is still valid JSON, just ASCII-only" —
  is enforced by both the unit test in `json.rs:73-84` (roundtrip via
  `serde_json::from_str` recovers the original `Value`) and by the two
  call-site tests that parse the header back and check the unicode
  values are intact.
- Surrogate-pair handling is correct: `ch.encode_utf16(&mut [0; 2])`
  at `json.rs:31` iterates the code units, so e.g. 🚀 (U+1F680) emits
  `\ud83d\ude80` rather than a single out-of-range escape — and the
  test at `json.rs:79` covers exactly that case.
- Scope is tight: the formatter only overrides `write_string_fragment`
  and inherits everything else from the default formatter, so map keys,
  array framing, number formatting, and escaping of `"`/`\\`/control
  chars all remain unchanged. The contract "this is the same JSON, just
  ASCII-escaped" is structurally preserved.
- Test renames the workspace path itself to `repo-東京` rather than just
  putting unicode in a value — that's the higher-value coverage because
  workspace paths are the field most likely to carry non-ASCII in the
  wild (cwd from the user's filesystem) and they're the keys of the
  `workspaces` map, not just values.

## Concerns / asks

- The fragment-scanning loop at `json.rs:18-37` reslices `fragment.as_bytes()`
  using `char_indices`. For an all-ASCII fragment this does one
  `write_all` of the whole slice via the `if start < fragment.len()`
  branch — fine. For a fragment that's mostly non-ASCII this becomes
  one syscall per character. Not a correctness issue, but worth a
  comment that the hot path is the all-ASCII one and non-ASCII is
  expected to be rare; otherwise a future maintainer might be tempted
  to "optimize" by buffering and break the formatter contract.
- `String::from_utf8(bytes)` at `json.rs:53` can in principle fail if
  some other formatter override later writes non-UTF-8 bytes. Today
  the only override is `write_string_fragment` and it only writes
  ASCII (either pass-through-of-ASCII or `\\u` escapes), so the
  `unwrap`/`map_err` is dead in practice. Worth either an
  `unwrap_or_else(|_| unreachable!())` with a comment, or leaving as-is
  with a one-liner explaining why it's defensively kept.
- The `merge_responsesapi_client_metadata` call at `turn_metadata.rs:88`
  re-encodes the merged map after `or_insert_with` — fine, but the
  reserved-fields test at `turn_metadata_tests.rs:145-160` only exercises
  the merge path with an `origin` entry. A test that has the *caller*
  try to overwrite a reserved field (e.g. `session_id`) with a non-ASCII
  value would prove that `or_insert_with` continues to win and that the
  stored ASCII-escaped form of the *server* value (not the caller's
  unicode value) ends up in the header. The current test asserts
  `json["session_id"].as_str() == Some("session-a")` but the caller
  also supplies `"client-supplied"` (ASCII), so the unicode-overwrite
  case isn't exercised.
- No changelog entry / no comment in `turn_metadata.rs` pointing at *why*
  the helper is used. A future drive-by refactor could revert the two
  call sites back to `serde_json::to_string` thinking it's a stylistic
  difference. Worth a one-line `// HTTP headers must be ASCII; see
  to_ascii_json_string` at each call site.

## Verdict

**merge-after-nits** — correctness is solid and the tests cover the
right invariants (ascii-ness + roundtrip equality + workspace-path
key escaping + surrogate pairs). The asks are all about future-proofing:
a comment at the two call sites explaining the ASCII requirement, and
ideally one extra test that proves a unicode caller value can't
shadow a reserved field's ASCII value.

## What I learned

`serde_json` deliberately omits a `ensure_ascii` flag (Python's
`json.dumps` has one), so the way to get ASCII-only JSON in Rust is
to plug a custom `Formatter` that overrides exactly
`write_string_fragment` — overriding fewer methods means fewer ways
the formatter can drift out of sync with the default's escaping
contract. The `encode_utf16` trick is the standard escape recipe and
handles supplementary-plane codepoints correctly via surrogate pairs.
