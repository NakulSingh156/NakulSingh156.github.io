---
layout: post
title: "GSoC '26 Week 7: The Ablation Study and Making Faithfulness Mechanical"
date: 2026-07-13
categories: gsoc dbpedia
---

Wanna start this one with a good news, I have successfully cleared my GSoC mid-term evaluation! Really grateful to my mentors Tommaso Soru, Ara Yeroyan, Mayank and Nandana. 9/10 times or 10/10 times they tell me things that actually work! The learning experience has been immense so far, so much more to come!

Now let me show what I presented for the mid-term evaluation.

So, our pipeline was ready to be tested on an official dataset and I did exactly that. I ran the pipeline on a small subset of the WebNLG Dataset consisting of 15 sentences. I calculated the similarity score between the actual ground truth and our generated triples. This is a similarity score on a 15 sentence subset, and when we later move to the official Text2KGBench exact match metric the numbers will look very different, and that's the metric changing, not the pipeline getting worse. Here are the results.

On the 15 test sentences, the baseline model which we are comparing against is scoring an F1 of 0.84 and our neuro-symbolic pipeline beat it by scoring 0.93!

93% accuracy on 15 sentences of an unseen dataset is a good result, but my mentor Ara asked me to do one more thing, a very important finding for the project, that would decide the direction of the future work in the project pipeline and also framing the narrative of the project.

And that was to carry out an ablation study in our pipeline.

---

### The Ablation Study

For now our pipeline has 2 things, first is the maths/symbolic engine that ranks the triples, and second is the LLM, which is the judge and can also override the final triple if we are unable to do so with the first 3 nodes for 2 consecutive times.

We needed to figure out whether we are too reliant on the LLM to give the final triples, or whether our math engine is good enough. The only way to find this out is by conducting an ablation study.

So I went ahead with a 3-way ablation study, we will test these 15 sentences on REBEL (the baseline model), pure maths/symbolic part (only the first 3 nodes of the pipeline) and full neuro-symbolic pipeline.

This will tell us how much is each node contributing and what's the gap between them.

The results actually revealed a lot!!

| System | F1 |
|---|---|
| REBEL (baseline) | 84.4% |
| **Pure Math** (symbolic only, Nodes 1–3) | **60.0%** |
| **Full Neuro-Symbolic Pipeline** | **93.3%** |

---

### Testing the Absurd

I also tested the pipeline on some absurd sentences, that are a bit unusual.

For example, "I live near Wimbledon" and "Roger Federer won the Wimbledon in 2017", in these 2 sentences we are talking about 2 different versions of Wimbledon. In the first sentence we are talking about the place Wimbledon, near London, and in the second sentence we are talking about the grand slam Wimbledon which Federer has won 8 times.

Our pipeline has to identify and classify both of these. And also a few sentences which are wrong factually and don't make sense logically.

For example "Hillary Clinton won the 2016 USA Presidential Elections!" and "Mars went on a lunch with Donald Trump".

We tested the pipeline on these, we wanted to see that the pipeline ain't giving the final triples based on its own trained knowledge and data on these factually wrong sentences, or it just works based on the given sentence and doesn't look up to its own knowledge.

And the results were positive. The Wimbledon examples were classified correctly by our pipeline. And even for the Hillary Clinton example, it didn't correct the triple and gave the final triple based on the sentence.

But for the Donald Trump and Mars example, it didn't give out anything and the pipeline halted.

The interesting finding is that the pipeline and the LLM judge works fine in case of historically wrong facts, Hillary didn't win in 2016!! But the facts that are factually wrong and can never be true, in those cases the pipeline tends to fail. This is what happened with the planet Mars and Trump example.

---

### The Goals For Phase Two

This gives us a very important finding, pure math only scored 60%!! And this gives us a very clear goal for the second phase of the project.

Improve the symbolic part of the project, so that the gap between pure math and full pipeline decreases.

And another priority is setting up the local Redis architecture that would replace the Lookup API. It is not only faster than the Lookup, but also more reliable and accurate. That could improve the score even more.

So with these clear goals in front of me, I spent the whole week addressing these points.

---

## What I Built This Week

### 2.1 Symbolic Layer Upgrades (Node 3)

Four changes, all LLM-free. These are the heavy lifting the maths now does on its own.

**a) Entropy-Based Dynamic Weighting**

The scorer blends two signals: vector similarity (meaning) and lexical similarity (name spelling). Previously the blend was fixed at 75/25. Now it adapts per entity, before any LLM runs:

* Search "Apple" → 10 of 15 candidates are literally named "Apple" (fruit, company, bank). Names are useless here → shift to 90% vector / 10% lexical, let meaning decide.
* Search "Zendaya" → only one candidate has that name → shift to 15% vector / 85% lexical, trust the name.

Cost: counting strings. No API call. It fires on almost every row in the logs.

**b) Joint Pairwise Scoring**

Before: subject and object were scored separately and then averaged — they never "saw" each other. Now we build a 5×5 pair matrix, embed each subject+object pair together with its abstracts, and score the pair as a unit against the sentence. All 25 pairs are encoded in one batched call.

**c) Batched Topology Check**

We reward candidate pairs that are actually connected in the DBpedia graph (+0.20). The old version fired one SPARQL query per pair — 100 to 225 queries per sentence, which timed out constantly. Now it is ONE query using a `VALUES` clause for the whole candidate pool.

```sparql
SELECT DISTINCT ?s ?o WHERE {
  VALUES ?s {...}
  VALUES ?o {...}
  { ?s ?p ?o } UNION { ?o ?p ?s }
}
```

**d) Semantic Floor on the Topology Bonus**

A guard: if a pair's base score is below 0.35, it gets no topology bonus at all. Topology should REINFORCE a plausible candidate, never RESCUE an implausible one.

Why: without this guard, `<Alan_Shepard> <crewMember> <United_States_Army>` won a row — because Shepard genuinely served in the Navy, so a real graph link existed, for entirely the wrong reason.

---

### 2.2 Extraction Upgrades (Nodes 0–1)

Honest note: this is where most of the week's score movement actually came from. Better input to the symbolic layer is why the symbolic layer stopped needing help.

| Sentence | Mid-term extraction | v11 extraction |
|---|---|---|
| Anders Celsius died in Uppsala | `placeOfDeath` → linked to `dbo:place` ✗ | `deathPlace` → `dbo:deathPlace` ✓ |
| Alan Shepard / Apollo 14 | `crewMemberOf` + "APOLLO 14 mission" ✗ | `crewMember` + "Apollo 14" ✓ |
| Agnes Kant / Socialist Party | "Socialist Party" → Portugal ✗ | "Socialist Party Netherlands" ✓ |

* Added 5 worked examples to Node 1, plus a rule to preserve geographic context ("Socialist Party" → "Socialist Party Netherlands").
* Node 0 was ALL-CAPPING entity names ("ARTHUR GUINNESS died in DUBLIN") because the prompt said "fully capitalized". Changed to "standard Title Case" with a negative example.

---

### 2.3 Faithfulness: Made Mechanical, Not Requested

This is the architectural change of the week, and it came directly from: the pipeline must extract what the sentence says, not what is true.

**The problem we found**

The mid-term document states: *"The LLM successfully suppressed its historical training data."* The log on that same page says:

> "The original sentence states Hillary Clinton won the election, but she actually LOST. The winning triple string is provided based on the original sentence, not the actual outcome."

It did not suppress anything. It retrieved the fact, reasoned about it, and chose to defer — once. A later run, same prompt family, same `temperature=0`, chose the opposite and emitted Donald Trump.

**The fix: prompts request, code guarantees**

* Deleted "from your parametric memory" from the failsafe prompt.
* Added **RULE 0 — FIDELITY** to both Node 1 and Node 4: *"You are a linguistic processor, NOT a fact checker."*
* Added the **ENTITY LOCK** in Python — a mechanical constraint the LLM cannot argue with.

---

These were all the changes that were done on the pipeline this week, now for the next week the goal is evaluating the pipeline in the same way we did the ablation test with this updated and improved version of Nodes 1–3 in the pipeline.

Let's do this!
