# ollama/ollama PR #15765 — Document direct GGUF pull from HF Hub

- **Repo:** ollama/ollama
- **PR:** [#15765](https://github.com/ollama/ollama/pull/15765)
- **Head SHA:** `ceedbc9c393a7f7e382cbfd500912336b6a30871`
- **Author:** rudi193-cmd
- **Size:** +50 / −0 across 1 file (`docs/import.mdx`)
- **Reviewer:** Bojun (drip-27)

## Summary

Pure docs PR. Adds a "Pulling a GGUF model directly from Hugging Face
Hub" section to `docs/import.mdx`, covering three sub-flows:

1. The `ollama pull hf.co/owner/repo:tag` direct-pull pattern.
2. Aliasing a hub-pulled model to a shorter local name with
   `ollama cp`.
3. Customising the system prompt / parameters by emitting a
   `Modelfile` with `FROM hf.co/...`.

Inserted between the existing "create from Modelfile" and "Quantizing
a Model" sections at line 110. No code changes, no behaviour changes.

## Key changes

### `docs/import.mdx:108–158` — new section

Three subsections, each with a copy-pasteable shell or Modelfile
snippet:

```shell
ollama pull hf.co/TheBloke/Mistral-7B-Instruct-v0.2-GGUF:Q4_K_M
```

The example chooses a real, well-known model (`TheBloke/Mistral-7B-
Instruct-v0.2-GGUF`) which is a good pedagogical pick — readers can
copy-paste it and verify the pattern works against an unambiguously
real Hub repo, rather than against an `owner/repo` placeholder that
they have to mentally substitute.

The aliasing flow:

```shell
ollama pull hf.co/owner/repo:Q4_K_M
ollama cp hf.co/owner/repo:Q4_K_M mymodel
ollama run mymodel
```

…and the Modelfile customisation flow:

```dockerfile
FROM hf.co/owner/repo:Q4_K_M

SYSTEM """
You are a helpful assistant specialised in <domain>.
"""

PARAMETER temperature 0.7
PARAMETER num_ctx 8192
```

The `<domain>` angle-bracket placeholder is the standard "fill this
in" convention; the `0.7` / `8192` parameter values are reasonable
defaults rather than placeholders, which keeps the snippet runnable
as-is.

## What's good

- The `hf.co/` direct-pull pattern is widely used in practice but
  was genuinely missing from the import guide. Documenting the
  most-common-modern workflow alongside the older convert-locally
  flow is the right structural call. The PR body's framing ("train
  → export GGUF → upload to HF Hub → `ollama pull hf.co/...`") is
  exactly the workflow people are doing, so it's worth surfacing
  in the docs that map to it.
- Three sub-flows in one section is the right granularity. Just
  the `pull` command would be terse-but-incomplete; including the
  `cp` aliasing and the `FROM hf.co/...` Modelfile flow turns a
  single-step example into a story about what a fine-tuner's local
  workflow actually looks like.
- The example model in the direct-pull section
  (`TheBloke/Mistral-7B-Instruct-v0.2-GGUF:Q4_K_M`) is a real,
  long-lived Hub repo. Concrete examples with real model names
  age better than `owner/repo:tag` placeholders because readers
  can verify the command actually works.
- `Q4_K_M` as the pulled quant tag in every example is a good
  default — it's the most popular quant level and most GGUF
  uploaders publish it. A reader copying these examples is most
  likely to find a working tag.
- The escaping of `Q4\_K\_M` in the prose (the markdown `\_`)
  is correct for `.mdx` — without the escape, the underscores
  would italicise the `K` and visually mangle the tag.

## Concerns

1. The aliasing example uses `hf.co/owner/repo:Q4_K_M` as the
   model spec but never tells the reader what to substitute. A
   one-line "where `owner/repo` is the Hub repo path" sentence
   below the snippet would close the loop for readers who skipped
   the prior section. This is a minor nit — the surrounding
   context makes it inferable — but docs PRs are exactly where
   "obvious from context" tends to fail for first-time readers.

2. No mention of authentication for gated/private Hub repos. The
   `hf.co/` prefix works for public repos out of the box, but
   gated models (Llama 3.x, etc.) require an `HF_TOKEN` env var
   or equivalent. For a docs section explicitly targeting
   "fine-tuned models published to Hub", the gated case is going
   to be a common stumbling point. Worth a one-line note: "For
   gated or private repos, set `HF_TOKEN` before pulling."

3. No mention of what happens when a Hub repo has *multiple*
   GGUF files (very common — quants are often packaged as
   separate files in one repo). The `:Q4_K_M` tag selector in
   the example implicitly answers this, but it's not stated
   that the tag must match a quant suffix in the Hub repo's
   filenames. Worth a sentence: "The tag after the colon
   selects which GGUF file in the repo to pull (e.g. a repo
   containing `model-Q4_K_M.gguf` and `model-Q8_0.gguf` exposes
   `Q4_K_M` and `Q8_0` as tags)."

4. The Modelfile snippet uses `num_ctx 8192` but doesn't note
   that this needs to be ≤ the underlying model's context limit
   to take effect (and that exceeding it is a silent capping).
   Probably out of scope for an import-flow doc, but worth a
   note since the value 8192 is wrong for some smaller / older
   models the reader might pull.

5. No example of the *failure* case — what error message does
   `ollama pull hf.co/...` emit when the repo doesn't exist or
   the tag doesn't match a file? Showing the canonical error
   would help readers self-diagnose typos. Optional enhancement.

## Risk

Zero behaviour risk — single docs file, no code or config.

## Verdict

**merge-as-is**

Documents a real and frequently-used workflow that was missing from
the import guide, with realistic examples and correct mdx escaping.
The five concerns above are all "nice to have" expansions; none of
them block merging this version, and the `HF_TOKEN` note (concern
2) would be a fine standalone follow-up PR rather than a blocker
on this one.

## What I learned

The pattern of "use a real, long-lived public repo as the example
even though you could use a placeholder" pays off in copy-pasteable
docs. Readers verify their setup works by running the example
verbatim before adapting it; if the example uses
`owner/repo:tag`, the only way to verify is to pick *some* repo
yourself, which doubles the cognitive load. Pinning to a
well-known third-party publisher (TheBloke here) is a small
maintenance bet that pays off in pedagogical clarity.
