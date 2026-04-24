# sst/opencode PR #24196 — feat(console): improve Polish translations

- **Repo:** sst/opencode
- **PR:** [#24196](https://github.com/sst/opencode/pull/24196)
- **Head SHA:** `6eb881ae3ce85f406c720b55bfc18c7522143a0b`
- **Size:** +12/-12 in `packages/console/app/src/i18n/pl.ts`
- **Reviewer:** Bojun (drip-23)

## Summary

Single-file translation polish for the opencode marketing/console UI
(`packages/console/app/src/i18n/pl.ts`). Twelve string changes, all in
the same dict object. No code change, no schema change, no key add or
removal — every edit replaces an existing Polish translation with a
slightly more idiomatic phrasing. Low-risk by construction.

## What's changed

Twelve string-value swaps (left = pre, right = post), grouped by
intent:

### "Free" → "without fees" framing (lines 9–10, 27–28 of diff)

- `nav.getStartedFree`: "Zacznij za darmo" → "Zacznij bez opłat"
  ("for free" → "without fees"). Slightly more formal; "za darmo" is
  perfectly idiomatic but reads as casual.
- `home.hero.subtitle.a`: "Darmowe modele w zestawie..." → "Dołączone
  bezpłatne modele..." ("Free models included" → "Included free-of-
  charge models"). Same swap of "darmowe" (colloquial) →
  "bezpłatne" (formal).

The style trade-off here is real — Polish marketing copy generally
splits on this register. The fact that `home.hero.subtitle.a` keeps
"bezpłatne" but the pricing pages keep `Darmowe` (e.g., line 72 of
diff: `"go.graph.free": "Darmowe"` is unchanged) means the codebase now
has **mixed register** between hero and pricing surfaces. Either commit
to "bezpłatne" everywhere or keep "darmowe" — but the inconsistency is
worse than either choice.

### "Role" → "Position" (lines 18–19)

- `error.roleRequired`: "Rola jest wymagana" → "Stanowisko jest
  wymagane". "Rola" maps directly to "role" (RBAC sense); "Stanowisko"
  maps to "position / job title". **This is a semantic regression** if
  the field is RBAC role selection — the user expects "Owner / Admin /
  Member" semantics, not "Senior Engineer / Manager". Worth confirming
  with the maintainers what UI string `error.roleRequired` actually
  guards. If the form field is labelled "Stanowisko" in pl.ts, fine;
  if it's still labelled "Rola" elsewhere, this introduces a
  copy/error-message mismatch.

### Verb polish (lines 36–37, 47–48)

- `home.faq.a1`: "pomaga pisać i uruchamiać kod" → "pomaga tworzyć i
  uruchamiać kod" ("helps write and run code" → "helps create and run
  code"). Marginal; "tworzyć" is broader and probably better aligns
  with "agent" framing.
- `home.faq.a3.p3`: "Chociaż" → "Choć" (both mean "although"; "Choć"
  is shorter and slightly more literary). Pure style.

### Word order tweaks (lines 48–49, 70–71)

- `home.faq.a3.p4.beforeLocal`: "Możesz nawet podłączyć swoje" →
  "Możesz podłączyć też swoje" ("You can even connect your" → "You
  can connect your too"). The new phrasing is **weaker** — "even" /
  "nawet" is doing rhetorical lifting in the original ("we go this
  far"); replacing it with "też" / "too" loses that. I'd revert this
  one.
- `go.banner.text`: "Kimi K2.6: limit użycia zwiększony 3× do 27
  kwietnia" → "Kimi K2.6: 3x większy limit użycia do 27 kwietnia"
  (passive → active). The active form reads better in a banner. Good
  swap.

### Pricing micro-copy (lines 65–66)

- `go.cta.promo`: "$5 pierwszy miesiąc" → "$5 za pierwszy miesiąc"
  (adds the preposition "za" / "for"). The pre-PR form is grammatically
  acceptable but feels clipped. Good swap.

### Genitive/case fixes (lines 56–57, 88–91)

- `zen.faq.a2`: "...nie używasz noża do masła do krojenia steku..." →
  "...nie używasz noża do masła, żeby kroić stek..." Replaces a
  double "do" + genitive "steku" with a "żeby + infinitive + accusative
  stek" construction. The original chained two `do` prepositions
  awkwardly ("knife of butter to cut of steak"); the new form reads
  natively.
- `workspace.usage.breakdown.cacheRead/cacheWrite`: "Odczyt Cache" →
  "Odczyt z cache", "Zapis Cache" → "Zapis do cache". Adds prepositions
  (`z` = from, `do` = to) and lowercases "cache". Both changes are
  correct Polish — noun-noun compounding without prepositions is an
  English calque that doesn't survive in Polish. The lowercase "cache"
  also matches Polish technical writing style.

### BYOK rebrand (lines 79–80)

- `workspace.providers.title`: "Przynieś własny klucz (BYOK)" → "Własny
  klucz API (BYOK)" ("Bring your own key (BYOK)" → "Own API key
  (BYOK)"). Loses the literal "Bring" framing but gains "API" as a
  qualifier — important because just "klucz" / "key" is ambiguous in a
  console UI (could be a license key, encryption key, etc.). Net win.

## Concerns

1. **Mixed register** — see "darmowe" vs "bezpłatne" above. Either
   commit fully or revert to original. Maintainers should pick a style
   guide and apply uniformly across `pl.ts`.

2. **`error.roleRequired` semantic shift** — "Rola" → "Stanowisko" is
   not a polish, it's a different word. Need confirmation that the form
   actually collects job title and not RBAC role. If RBAC, this should
   be reverted.

3. **`home.faq.a3.p4.beforeLocal` weakened** — losing "nawet" / "even"
   removes the rhetorical hook the original copy was leaning on.

4. **No screenshots** — for a translation PR touching marketing copy
   that ships verbatim to users, a single before/after screenshot of
   the hero section would catch any line-wrap or layout regression
   from the longer phrasings. "Dołączone bezpłatne modele lub podłącz
   dowolny model od dowolnego dostawcy," is noticeably longer than
   "Darmowe modele w zestawie lub podłącz dowolny model od dowolnego
   dostawcy," and may push past a 2-line cap on smaller viewports.

5. **No native-Polish reviewer signal** — translation quality calls
   are subjective; a one-line "+1 from a native PL speaker" in the PR
   thread from a different contributor would be helpful before merge.
   Without that, the maintainer is taking the contributor's word for
   it on a register-sensitive language.

## Verdict

`merge-after-nits` —

- revert the `error.roleRequired` change unless confirmed to be a job
  title field, not RBAC;
- decide on register ("darmowe" vs "bezpłatne") and apply uniformly;
- consider reverting `home.faq.a3.p4.beforeLocal` — losing "nawet" hurts
  more than the word-order swap helps;
- ask for a native-PL second opinion in the PR thread.

The other 8–9 changes are clean wins (preposition fills, case fixes,
BYOK qualifier, lowercase "cache"), and would be `merge-as-is` on
their own.

## What I learned

i18n PRs are a good reminder that "translation polish" often hides
semantic decisions: changing "Rola" → "Stanowisko" looks like a
synonym swap but is actually a different field type. The reviewer
heuristic to apply to every i18n PR: for each changed key, ask
"could a non-Polish-speaker reading this dict tell whether this key
is doing UI labelling, error-state messaging, or schema-aligned
field naming?" — and if not, treat the change as semantic, not
cosmetic.
