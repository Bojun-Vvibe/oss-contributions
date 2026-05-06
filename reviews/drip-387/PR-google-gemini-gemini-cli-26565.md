# Review: google-gemini/gemini-cli #26565 — fix(core): prevent isBinary false-positive on Windows PTY streams

- Carrier: `google-gemini/gemini-cli`
- PR: #26565
- Head SHA: `84db4b5d`
- Drip: 387

## Verdict
`merge-after-nits` (man)

## Summary
A real and well-targeted bug fix for Windows PTY output being misclassified
as binary by the shell-execution sniffer. On Windows, `node-pty` emits VT
control sequences (e.g., `OSC 0;<title>BEL` for the window title) that
legitimately contain embedded null bytes — the previous `isBinary(buf)`
implementation tested `for (b of sample) if (b === 0) return true`, so a
single null byte anywhere in the first 512 bytes flipped the entire stream
to `binary_detected` mode and the user's prompt prefix never made it to
the renderer.

Fix: extend `isBinary(data, sampleSize=512, isPtyOutput=false)` with a new
opt-in PTY mode that (a) strips ANSI/VT escape sequences via a new
`stripAnsiFromBuffer(data: Buffer): Buffer` helper, then (b) uses a
null-byte *ratio* (>10%) instead of a single-byte trigger. The single
caller that was misclassifying — `ShellExecutionService` at
`shellExecutionService.ts:1137` — now passes `isPtyOutput: true`.

## Diff anchors
- `packages/core/src/utils/textUtils.ts:25-95` — new `stripAnsiFromBuffer`
  implementation. Walks bytes manually (not regex on string — important
  because the input is `Buffer` and decoding could itself corrupt
  partial UTF-8 sequences). Handles three sequence shapes:
  - **CSI** (`ESC [ ... <0x40-0x7E>`): scans until a final byte in the
    range and skips it. Correct per ECMA-48 §5.4.
  - **OSC** (`ESC ] ... (BEL | ESC \\)`): scans until `0x07` (BEL) *or*
    `ESC` followed by `0x5C` (string terminator). The BEL form is the
    Windows-PTY case the bug is about; the ESC-`\\` form is the
    canonical form per VT100.
  - **Two-byte**: `ESC <0x40-0x5F>` covered by the `next >= 0x40 && next
    <= 0x5f` branch. This subsumes simple sequences like `ESC D` (Index)
    and `ESC E` (NEL).
  - Lone trailing `ESC` (no following byte) is dropped.
- `:96-118` — new `isBinary(data, sampleSize, isPtyOutput=false)`
  signature with default `false` preserving the existing 4 call-site
  contracts (file readers, pipe sniffers, etc., still use strict
  null-byte mode). When `isPtyOutput=true`:
  1. Strip ANSI from the sample.
  2. If stripped is empty, return `false` (pure escape sequences are
     not binary).
  3. Count nulls in the stripped buffer; binary iff `nullCount /
     sample.length > 0.10`.
- `packages/core/src/services/shellExecutionService.ts:1137` — single-line
  caller change `isBinary(sniffBuffer)` → `isBinary(sniffBuffer, 512,
  true)`. Correctly placed at the PTY-output sniff path; not applied to
  any non-PTY shell path so file-redirect detection is unaffected.
- `textUtils.test.ts:200-340` — exhaustive test coverage:
  - `stripAnsiFromBuffer` block: 7 cases covering no-sequences, CSI,
    OSC-BEL, OSC-ST, two-byte, mixed, and pure-sequences-only.
  - `isBinary` default-mode block: 6 cases pinning the legacy contract
    (null/undefined/empty, plain ASCII, single null byte, all-null
    buffer, sample-size truncation, custom sample-size override).
  - `isBinary` PTY-mode block: 6 cases including the actual bug
    repro `'\x1b]0;\x00Window Title\x07\x1b[0mPS C:\\Users>...'`
    asserting `false`, plus the 90%-null worst case asserting `true`
    so genuine binary still trips the detector.

## Concerns / nits
1. **The 10% threshold is unjustified in code.** `nullCount / sample.length
   > 0.10` is a magic constant. A short comment explaining why 10% (e.g.,
   "real PTY output rarely has more than ~5% nulls from VT sequences;
   binary data on the order of >50% nulls is common; 10% is a safe
   midpoint") would prevent future drift.
2. **`stripAnsiFromBuffer` builds an intermediate `number[]` then
   converts via `Buffer.from(out)`.** For a 512-byte sample this is
   negligible, but for a hot path it would be ~5-10× faster to
   pre-allocate a `Buffer.alloc(data.length)` and write into it with a
   write-index. Pure micro-perf nit, not worth blocking on.
3. **`stripAnsiFromBuffer` doesn't handle DCS / SOS / PM / APC
   sequences** (`ESC P`, `ESC X`, `ESC ^`, `ESC _`), which are also ST-
   terminated. Unlikely to fire in real Windows PTY output (these are
   typically used by remote-graphics or sixel renderers) but worth a
   `// TODO: DCS family` comment.
4. **`isPtyOutput` is a positional boolean.** The signature
   `isBinary(data, sampleSize=512, isPtyOutput=false)` invites bugs
   like `isBinary(buf, true)` (where `true` coerces to `1` and becomes
   the sample size). An options-object signature
   `isBinary(data, {sampleSize, isPtyOutput} = {})` would be safer but
   would touch the 4 existing callers — defensible to defer.
5. **No regression test against the actual `isBinary` consumer
   (`ShellExecutionService`).** The unit-level coverage is
   excellent but a single integration test feeding a captured Windows
   `node-pty` byte sequence through `processNode` would lock the
   end-to-end fix.

## Risk
Low. The default `isPtyOutput=false` parameter preserves existing
behavior at every other call site verbatim (file readers, pipe sniffers,
diff binary detection). The single behavioral change at the PTY-sniff
site is well-tested and the 10%-null threshold is conservative enough
that genuine binary output (PNG headers, ELF files, etc.) still trips
correctly.
