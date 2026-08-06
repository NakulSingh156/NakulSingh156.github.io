---
layout: post
title: "GSoC '26 Week 8: Scaling Up and the Multi-Triple Problem"
date: 2026-07-20
categories: gsoc dbpedia
---

Last week I ended by saying the next goal is to evaluate the improved pipeline the same
way we did the ablation test. So that is exactly what I did this week, and scaling up from
15 sentences to the real thing revealed a problem I did not see coming.

### Moving to Text2KGBench

Till now we were testing on a 15 sentence subset of WebNLG. That was a smoke test, not a
benchmark. 15 sentences means one row is worth 6.7 points, so the confidence interval was
roughly ±13 points. You cannot make any real claim on that.

So this week we moved to the official **Text2KGBench DBpedia-WebNLG** task. Here is what
it actually is:

* 19 domains, each with its own ontology
* 2,014 test sentences
* 6,259 gold triples
* The official metric is **exact triple match**, macro-averaged over the 19 domains

This is a much harsher metric than the similarity score we were using. A triple that is
semantically perfect but writes `Perth,_Australia` instead of `Perth` scores a straight
zero. No partial credit.

And here is what we are being compared against, these are the published numbers on the
same task:

| System | Macro F1 |
|---|---|
| REBEL (zero-shot) | 0.060 |
| T5-Large (fine-tuned) | 0.389 |
| GPT-3.5 Turbo, 5-shot | 0.510 |
| GPT-4o, 6-shot | 0.570 |
| **NEF (the paper we are compared to)** | **0.628** |

### Building the harness

Before running anything I built `text2kg_harness.py` — the loader plus a scorer that
reproduces the official metric exactly.

And I did one thing here that I want to highlight, because it saved me later. I scored the
**gold against itself**. If the scorer is correct, gold vs gold must give exactly 1.0 on
every domain.

MACRO: P=1.0000 R=1.0000 F1=1.0000 -> 1.0000 across the board means the scorer reproduces the official metric.

Without this check I would have had no way to know whether a low score meant a bad
pipeline or a bad scorer. Verify the ruler before you measure anything with it.

### The problem nobody warned me about

Then I ran the pipeline on the full benchmark and the scores were terrible. Like 0.1
terrible.

I went digging and found the reason immediately, and it was structural, not a bug.

**Our pipeline emitted exactly ONE triple per sentence.**

Text2KGBench sentences average **3.11 gold triples each**:

* 68.9% of sentences need 3 or more triples
* only 8.1% need exactly 1
* and **49.3% change SUBJECT between triples** — the object of triple 1 becomes the
  subject of triple 2

So even a PERFECT single-triple pipeline is capped. Here is the arithmetic:

recall = 1 / 3.11 = 0.322 F1 = 2(1.0 × 0.322) / (1.0 + 0.322) = 0.487 <- HARD CEILING

**0.487.** That is below the GPT-4o baseline of 0.570 and nowhere near NEF's 0.628. Even
if every single triple we emitted was perfect, we could not compete.

Multi-triple was not an improvement we could add later. It was the entry fee.

### The multi-triple refactor

So I refactored the pipeline. Node 1 now returns a **LIST** of triples instead of one.
Nodes 2 → 3 → 4 then run once per extracted triple, each with its own retry budget and its
own entity lock anchor.

The good part: Nodes 2, 3 and 4 did not need to be rewritten at all. They always operated
on a single (subject, predicate, object). They just run N times now.

I also had to teach the extractor about **entity chaining**, because half the sentences do
it. A sentence like "X was designed by Y, who was born in Z" needs the object of triple 1
to become the subject of triple 2. That went into the Node 1 prompt with worked examples.

### Early results

| Stage | 3_airport |
|---|---|
| Single-triple pipeline | 0.107 |
| + multi-triple + ontology bounding | 0.314 |
| + faithful SPARQL gate | 0.419 |

Also fixed along the way: the ontology loader was only reading `ObjectProperty` and
completely ignoring `DatatypeProperty`. Fixing that took the loaded property set to
**2,865 properties** (1,105 object + 1,760 datatype). Every date, every number, every
measurement was previously unreachable.

### Next Steps

The pipeline now emits multiple triples and the scores are moving, but 0.419 is still a
long way from 0.628. Next week I go bug hunting properly, domain by domain. Stay tuned!
