---
layout: post
title: "GSoC '26 Week 11: The Full Sweep, Three Autopsies, and the Bug That Cost Two Domains"
date: 2026-08-10
categories: gsoc dbpedia
---

This is the week the project got its number. It is also the week I learned that a
surprising score is a bug report until you prove otherwise.

### The first full sweep

19 domains, 2,014 sentences, ~16 hours unattended. I ran it in two batches because of a
Jupyter quirk worth mentioning: `IOPub data rate exceeded` kills the *display*, not the
kernel. The run continued fine and every chunk saved to disk. I lost the log, not the work.

**Macro F1 = 0.598 across all 19 domains.** Above my optimistic band. `6_politician` hit
0.8074, `18_scientist` 0.7979.

And three results looked wrong to me:

- `13_food` had scored **0.66** on my Mac in an earlier run. Here it was **0.4883**.
- `2_musicalwork` finished at 0.5173 with **70 zero-scoring sentences out of 209**.
- `19_film` at 0.4754, with zeros clustered oddly.

I could have shipped 0.598 and moved on. Instead I did what my mentors and I agreed to
call an autopsy: dump every failing sentence with its gold, its prediction, and for
food, the same sentence's prediction from the earlier, better run. **All of it offline,
against saved JSON, costing zero API calls.**

### Autopsy 1: the bug that cost two domains

The food diff was unambiguous. **76 sentences dropped by more than 0.10; only 20 improved.**
That ratio kills the "run variance" theory immediately. And the failures were absurd:

```text
SENT:  Bionico is a food found in Mexico from the region Jalisco.
GOLD:  Bionico | country | Mexico          <- entity URI
MAC:   Bionico | country | Mexico          ✓
SRV:   Bionico | country | "Mexico"        <- quoted literal
```

The fact is correct. The country is correct. The **format** is wrong, so it scores zero.

The cause took a while to find and I still wince at it. Text2KGBench tells you, per domain,
which values are written as quoted text and which as entity links and the conventions
differ between domains. `12_monument` quotes `country`. `13_food` writes it as a URI.

My code collected those per-domain rules into a **global set that was never reset between
domains in the same process.** Domain 12 ran, taught the pipeline "country is quoted," and
every domain after it inherited that. Food ran right after monument.

**127 gold triples destroyed in `13_food`.** Then I checked the whole benchmark for the same
pattern and found the silent second victim: **`14_writtenwork`, 77 `country` + 9 `almaMater`
triples, across 86 of its 127 sentences.** Its 0.5651 was never its real score and I had
no idea.

Nothing in the output looked broken. No error, no exception, no rejection. Just quotes
where there should have been angle brackets.

**The lesson: state that accumulates across runs is a data-corruption bug waiting to
happen, and it will not announce itself.** The fix is four lines, consult the current
domain's rules, never a global.

### Autopsy 2 and 3: the pipeline knew the answer and threw it away

`2_musicalwork` and `19_film` turned out to be the same disease.

```text
SENT:  Train's hit Mermaid was put out by the Sony Music Entertainment record label
GOLD:  Mermaid_(Train_song) | recordLabel | Sony_Music_Entertainment
PRED:  Mermaid              | recordLabel | Sony_Music_Entertainment
```

DBpedia separates same-named things with a bracket suffix. `Mermaid` is the sea creature.
`Mermaid_(Train_song)` is the song. My resolver looked up the bare word and took the most
popular match, so the whole triple scored zero even though the relation and object were
perfect.

**35 of musicalwork's 70 zeros. 39 of film's 45.** In film it was almost entirely one
entity: `It's_Great_to_Be_Young` where gold wants `It's_Great_to_Be_Young_(1956_film)`.

Here is the part that stung. Look at the sentence again. It *says* "Train." Film's
sentences say "the 1956 film" nearly every time, one of them literally contains the string
"(1956 film)" and we still dropped it. **The disambiguating evidence was sitting in the
input and my pipeline never used it.**

### The v14 fixes

Five changes, all validated offline against saved data before I spent a rupee of API
budget:

1. **Per-domain literal rules.** The leak, fixed at the source.
2. **Context-qualified sense lookup.** Node 2 now builds extra index queries out of the
   sentence's own words, `"Mermaid (Train song)"`, `"It's Great to Be Young (1956 film)"`,
   because page-title surfaces are first-class keys in my index, so a correct guess hits
   exactly. Node 3 then scores the base title without the bracket (so the qualified sense
   is not punished on string similarity) and adds a bonus scaled by how many qualifier
   words the sentence actually supports. `(1956 film)` beats `(film)` beats bare title.
3. **Index cross check before demoting.** An object missing from the local store but
   present in the 16.2M-mention index is a store coverage gap, not a fake entity. Keeps
   `FIMI` and `Crucial_Blast` as URIs; still demotes genuinely non-existent things.
4. **Symmetric-predicate swap guard.** `dishVariation` is stored both ways in DBpedia, so
   the inverted-exact check proves nothing there and was flipping correct triples.
5. **Literal polish.** Bare year under a `*Year` predicate becomes `YYYY-01-01`;
   `"£282,838"` becomes `282838.0`.

Nothing references a benchmark domain by name. Every fix is a generic code path.

Rerun of the four affected domains — 602 sentences, 4.5 hours, about $3:

| Domain | Before | After | Δ |
| :--- | ---: | ---: | ---: |
| `13_food` | 0.4883 | **0.7066** | +0.22 |
| `14_writtenwork` | 0.5651 | **0.7309** | +0.17 |
| `19_film` | 0.4754 | **0.7177** | +0.24 |
| `2_musicalwork` | 0.5173 | **0.6063** | +0.09 |

### Then my mentor Ara made me do it properly

At this point I had 0.6356, but it was **4 domains on the new code and 15 on the old**.
Ara's note was exactly right: that number is provisional and a mixed run should not be
cited. Freeze the pipeline at a tag, re run all 19 on that one version, and cite only that.

So I tagged `v14-final`, stopped touching the code entirely, and re-ran everything.

Two things happened during that clean sweep worth writing down.

**First, the first two domains came back *worse* and I panicked.** `1_university` 0.6086 →
0.58, `3_airport` 0.6338 → 0.6144. If every domain drifted down by 0.02–0.03 the macro
would land below the baseline. I nearly started debugging the fixes.

The actual cause was the host. `free -h` showed **5GB deep in swap with 223MB free** — and a
second, password-protected Redis instance from a system package was holding over a gigabyte
that my pipeline could never have used. I stopped it, and the next domains reproduced at
parity or better (`4_building` +0.020, `7_company` +0.011, `8_celestialbody` +0.016). The
drift was the machine, not the code.

I re measured the two degraded domains under a policy I wrote down *before* looking at the
results: identical host conditions for every domain, re measured values are the reported
ones, no cherry picking. Airport came back at 0.6051, **lower than its degraded run.**
I reported that number, because the rule was set in advance and the whole point of setting
it in advance is that you do not get to renegotiate afterwards.

**Second, 3 of 2,014 sentences returned empty predictions** from transient API failures.
Documented, not retried. Maximum possible macro impact: 0.001.

### The number

**Macro F1 = 0.6317**, clean sweep, one frozen pipeline version, one host environment. The
GPT-4o baseline I have been chasing all summer is **0.628**.

| Domain | NEF 2.0 (ours) | NEF 1.0 (GPT-4o) | Δ |
| :--- | ---: | ---: | ---: |
| `6_politician` | **0.7991** | 0.722 | +0.077 |
| `18_scientist` | **0.7732** | 0.561 | +0.212 |
| `9_astronaut` | **0.7377** | 0.730 | +0.008 |
| `14_writtenwork` | 0.7309 | 0.768 | −0.037 |
| `8_celestialbody` | **0.7233** | 0.707 | +0.016 |
| `19_film` | 0.7177 | 0.753 | −0.035 |
| `15_sportsteam` | **0.7103** | 0.679 | +0.031 |
| `13_food` | 0.7066 | 0.765 | −0.058 |
| `16_city` | 0.6853 | 0.747 | −0.062 |
| `7_company` | **0.6804** | 0.494 | +0.186 |
| `4_building` | **0.6667** | 0.648 | +0.019 |
| `2_musicalwork` | **0.6063** | 0.443 | +0.163 |
| `3_airport` | 0.6051 | 0.721 | −0.116 |
| `1_university` | 0.5935 | 0.604 | −0.011 |
| `5_athlete` | 0.5566 | 0.577 | −0.020 |
| `17_artist` | 0.5264 | 0.609 | −0.083 |
| `11_meanoftransportation` | 0.5014 | 0.523 | −0.022 |
| `10_comicscharacter` | 0.4000 | 0.519 | −0.119 |
| `12_monument` | 0.2820 | 0.370 | −0.088 |
| **MACRO** | **0.6317** | **0.628** | **+0.004** |

We lead the baseline on **8 of 19 domains**, including `2_musicalwork` by +0.163, their
single worst domain.

And the honest reading, which I put in my own report: I measured run-to-run variance at
**±0.02 per domain and ±0.005 macro**. Beating 0.628 by 0.004 is *inside* that. So the
claim is **parity with GPT-4o, not superiority.** That was not fun to write.

### Next Steps

Package it. Docker, deployment, and the reproducibility bundle my mentors asked for. Lets
go!
