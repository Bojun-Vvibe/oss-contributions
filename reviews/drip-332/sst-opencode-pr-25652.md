# sst/opencode PR #25652 — docs: Remove LSP as differentiator with Claude Code

- **Link:** https://github.com/sst/opencode/pull/25652
- **Head SHA:** `7a8625c6b156f92200ef8897158a3a071295084a`
- **Author:** scarf005
- **Created:** 2026-05-01
- **Files:** 22 README translations (`README.md`, `README.ar.md`, `README.bn.md`, `README.br.md`, `README.bs.md`, `README.da.md`, `README.de.md`, `README.es.md`, `README.fr.md`, `README.gr.md`, `README.it.md`, `README.ja.md`, `README.ko.md`, `README.no.md`, `README.pl.md`, `README.ru.md`, `README.th.md`, `README.tr.md`, `README.uk.md`, `README.vi.md`, `README.zh.md`, `README.zht.md`)
- **Diff size:** +1 / −22 (one bullet line removed per language; one English wording tweak)

## Summary
Removes the "Built-in opt-in LSP support" / "LSP support out of the box" bullet from the differentiator list across every translated README. The English `README.md` change at line 135 swaps `Built-in opt-in LSP support` for nothing — the bullet is dropped entirely.

## Specific citations
- `README.md:135` — bullet removed: `- Built-in opt-in LSP support` (English source).
- `README.zh.md:134`, `README.zht.md:134`, `README.ja.md:135`, `README.ko.md:135`, `README.de.md:135`, `README.es.md:135`, `README.fr.md:135`, `README.it.md:135`, `README.pl.md:135`, `README.ru.md:135`, `README.tr.md:135`, `README.uk.md:136`, `README.vi.md:135` — same bullet removed in each translation, no other content changed.
- `README.th.md:135` — removes the longer Thai phrasing `รองรับ LSP ใช้งานได้ทันทีหลังการติดตั้งโดยไม่ต้องปรับแต่งหรือเปลี่ยนแปลงฟังก์ชันการทำงานใด ๆ`.

## Observations
- 22-file change but every diff is mechanical and identical in shape (single-line bullet removal). No code, no test impact.
- The PR description rationale (the alternative tool now also ships LSP, so it is no longer a differentiator) is accurate and consistent across all locales.
- Translations are kept in sync — no language was missed.

## Nits
- The line above the removed bullet talks about provider-agnosticism. After removing the LSP bullet, the next bullet is "A focus on TUI". Flow still reads cleanly; no reordering needed.
- Consider whether LSP is still worth mentioning *somewhere* deeper in the README (under "Features" rather than "Differentiators") rather than dropping the marketing line entirely. Not blocking.

## Verdict
`merge-as-is`
