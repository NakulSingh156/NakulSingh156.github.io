---
layout: post
title: "GSoC '26 Week 10: The Model, The Index, and the Retrieval"
date: 2026-08-03
categories: gsoc dbpedia
---

Three big things this week, and one of them gave the biggest single jump of the entire
project. Also I was wrong about something and the data corrected me, which I will get to.

### 1. The model ablation

Tsoru suggested testing GPT-5.6 Luna, which is roughly 20x cheaper than GPT-4o. The
question we wanted answered: **how much of our gap is the model, and how much is our
pipeline?**

So I froze the pipeline completely and changed only the model string. Four domains, chosen
to span our whole range from best to worst.

| Domain | LLaMA-3.3-70B | GPT-5.6 Luna | Δ | real zeros |
|---|---|---|---|---|
| 13_food | 0.3960 | **0.6150** | **+0.219** | 26 → 11 |
| 4_building | 0.3404 | 0.4612 | +0.121 | ~20 → 18 |
| 18_scientist | 0.6129 | 0.6627 | +0.050 | — |
| 10_comicscharacter | 0.2269 | 0.2037 | **−0.023** | 21 → 22 |

Look at that ordering. It is not random, and it is not about the topic of the domain.

**Where the entities are findable in DBpedia, the model is the bottleneck.** Food gained
+0.219 and its zero-scoring sentences more than halved. Perfect sentences went 16 → 39.

**Where the entities are NOT findable, the model changes nothing.** Comicscharacter got
slightly WORSE despite a far stronger model.

That comics result is the important one. Luna is a much better reasoner than LLaMA. On
comics it bought literally nothing. That is only possible if the failure happens **before**
the reasoning even starts — no model can pick the correct candidate if the correct
candidate was never retrieved in the first place.

Two different reasons a domain flatlines: comics has no retrievable candidates, scientist
has no headroom left. Same number, completely different cause.

Also worth saying: Luna runs at roughly the same price as LLaMA and about 3x faster per
call. So the cost story survives completely.

### 2. The local surface-form index

This is the thing I have wanted to build since the mid-term. Replace the live DBpedia
Lookup API with a local Redis index built from the DBpedia dumps.

Built from `labels_en`, `redirects_en`, `disambiguations_en`, plus `wikilinks` for a
popularity signal.

* **16,161,334 surface-form keys, 27,849,395 candidates**
* Scored as `tier_weight × (1 + log10(1 + indegree))`
* Tier ordering dominant: labels > redirects > disambiguations, with popularity breaking
  ties WITHIN a tier

Now, I did not want to just assume my ranking was right. So I tested four different sort
keys against the benchmark:

| Sort key | rank@1 |
|---|---|
| **A: tier first, then popularity** | **97.4%** |
| D: labels+redirects merged | 95.1% |
| C: blended tier × popularity | 94.6% |
| B: popularity only | 93.4% |

Tier-first wins, and **why** it wins is the interesting part:

* `"New York"` → tier-first gives `New_York`. Popularity-first gives `New_York_City`,
  `New_York_(state)`, `New_York_Yankees` — the exact match does not even make the top 3.
* `"Asian South Africans"` → popularity-first ranks the more famous sibling
  `Indian_South_Africans` first.

Popularity-first answers "what is famous". Tier-first answers "what did the text say".

That is the whole thesis of this project sitting inside a sort key.

### 3. Per-sentence few-shot retrieval

Our Node 1 prompt had fixed few-shot examples, the same ones for all 19 domains. I replaced
the convention examples with **6 examples retrieved per sentence** from that domain's TRAIN
split, using BM25 with MMR diversification. The format examples stay fixed. Zero extra LLM
calls.

| Domain | before | after | Δ |
|---|---|---|---|
| 10_comicscharacter | 0.2037 | **0.4111** | **+0.207** |
| 4_building | 0.5352 | **0.6387** | +0.104 |
| 13_food | 0.6150 | **0.6616** | +0.047 |

Comics **doubled**. And the reason is a genuinely strange finding.

Gold for that domain uses entity names like `Arion_(comicsCharacter)`. I went looking for
that URI and it does not exist. Not in the dumps, not on live DBpedia, nowhere:

* **51 occurrences** of `comicsCharacter` inside Text2KGBench itself — gold, train split,
  even the prompt files
* **0 occurrences** across 29 million lines of DBpedia dumps
* **0 triples** on live DBpedia today

It is a WebNLG annotation artifact that was never a real DBpedia URI. No index in the world
can produce it. But retrieval CAN, because the convention lives in the train split, and the
train split is exactly where we are allowed to look.

The surprise was building gaining +0.104. The retrieved examples are not just teaching
naming — they are teaching **predicate choice and triple granularity**, which is where all
our "entities right, predicate wrong" failures lived.

**Building's full arc this week: 0.4501 → 0.6387.**

### Where I was wrong

I had a theory that our weak domains were weak because their entities are intrinsically
obscure. Then I got hold of NEF's per-domain table and it refuted me flatly — `13_food` is
their SECOND BEST domain at 0.765 and it was one of our worst at 0.396. If food entities
were intrinsically hard, they would struggle too.

The correlation between our per-domain scores and theirs is only about +0.45. The domains
we find hard are largely NOT the domains they find hard.

Which is actually good news. It means our weakness is a property of our pipeline, not of
the data, and pipeline problems can be fixed.

### The noise floor

One more thing worth reporting. I ran the same configuration twice and got a +0.028
difference with zero fixes applied. So **any single-run delta below about 0.03 is
meaningless**. That retroactively justifies rejecting several fixes I had tested at that
scale, and from now on the final numbers get 2-3 runs with mean and spread.

### Next Steps

The full 19 domain sweep on the server. Everything so far has been measured on 3 or 4
domains that I picked, and picked domains flatter you. The macro across all 19 is now the
biggest unknown in the project. Time to find out!
