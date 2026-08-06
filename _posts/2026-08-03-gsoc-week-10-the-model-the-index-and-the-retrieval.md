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
