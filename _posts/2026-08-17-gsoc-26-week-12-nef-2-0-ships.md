---
layout: post
title: "GSoC '26 Week 12: NEF 2.0 Ships — Docker, DBpedia's Repo, and Learning What 'Significant' Means"
date: 2026-08-17
categories: gsoc dbpedia
---

Last week produced a number. This week turned it into something someone else can run, put
it in DBpedia's official repository, and got the reality check that reframed the whole
project.

### The stack in a box

The benchmark result is one thing. "Anyone can pull this and extract triples" is another,
and my mentors were clear that the second one is what makes the work useful.

Three containers:

- **Redis** holding the 16.2M-mention surface-form index
- **oxigraph** serving the local DBpedia store read-only
- **the pipeline**, behind a single endpoint

```text
POST /extract  {"text": "The song Mermaid is by the band Train."}
→ {"triples": [{"sub": "Mermaid_(Train_song)", "rel": "artist", "obj": "Train_(band)"}]}
```

Every returned triple carries its verification status `verified` against DBpedia, or
`faithful-unverified` (the sentence asserts it, DBpedia does not have it). That
distinction has been the design principle of this pipeline since the beginning and it
finally shows up in the API where downstream users can filter on it.

Things I got wrong and fixed while building it:

- Configuration was hardcoded to `127.0.0.1`. Made it env-driven, with defaults that
  reproduce the benchmark exactly so the frozen result stays reproducible.
- Redis answers `LOADING` for about three minutes while it reads the 1.4GB index dump. My
  first compose file started the pipeline immediately and it looked like Redis was down.
  Now there is a healthcheck and the pipeline waits.
- Related, and this one cost me an hour of confusion: **a Redis that the pipeline
  auto-starts is a child of the Jupyter kernel, so it dies on every kernel restart.** It
  has to run as an independent daemon. That is now written into the deployment notes.
- Port 8000 was already taken on my server, so the first end to end test 404'd against a
  completely unrelated service. Documented the port override.

Then the moment that made the week: the deployed container, on a fresh host, against the
real index and store:

```text
Józef_Franciszek_Darzyn_Ciemiński | birthPlace | Borzyszkowy
Józef_Franciszek_Darzyn_Ciemiński | birthDate  | 1867-08-04
```

An obscure 19th-century Polish priest, from a real Wikipedia abstract, diacritics intact,
prose date normalised. Not a benchmark sentence. Just the system doing its job.

The image is public: `ghcr.io/nakulsingh156/neural-extraction-framework:v2.0`

### Getting the cost claim wrong, and fixing it

I had been saying "matches GPT-4o at ~1/20th the cost." My mentor Tommaso asked me to verify it,
because last year's run cost roughly a sixth of what I was quoting.

He was right and my claim was conflating two different things:

| Comparison | Ratio | What it actually measures |
| :--- | :--- | :--- |
| Per token | ~20× cheaper | $0.10 / $0.60 vs $2.50 / $10.00 per M tokens — just the price sheet |
| System vs system | ~3–4× cheaper | ≈$8–10 vs ≈$30–35 for the full 2,014-sentence benchmark |

The gap between those two ratios is the interesting part, and it is by design: **I make
8–12 LLM calls per sentence where NEF 1.0 makes 1–2.** All that judging, verification and
repair is affordable *because* the model is cheap. The correct sentence is "a cheap model
funds expensive verification," which is a better story than the one I was telling by
accident.

Every document is corrected. I would rather be the person who fixes a number in public
than the person whose numbers cannot survive a mentor's question.

### Into DBpedia's repository

Cleanup week, on his instructions: tidy the file tree, use the project's real name
everywhere, and stop working in a side repo.

The code now lives in `dbpedia/neural-extraction-framework/GSoC26` pipeline, index build
tooling, Docker stack, all 19 domains' per-sentence results, the reproducibility bundle
(code tag, model config, benchmark commit, the exact nine dumps behind the store), and the
full fix history.

My code, my results, in a globally recognised open source GSoC organisation's repository. I sat
looking at that page for a while.

### The scaling study

My mentor Tommaso gave a real assignment: how much time and money to extract RDF from
Wikipedia abstracts?

I did not want to hand him numbers from a blog post, so I counted the workload from the
DBpedia snapshot already loaded on my own server:

```sparql
SELECT (COUNT(*) AS ?c) WHERE { ?s rdfs:comment ?o }   # → 4,643,098
```

**4,643,098 English abstracts.** Then a 400 abstract sample for sentence statistics:
**2.26 sentences per abstract, ≈10.5 million sentences** for the full corpus.

At my measured unit cost, phased with a review gate between every step:

| Phase | Scope | Cost | Gate to proceed |
| :--- | :--- | ---: | :--- |
| 0 | 100 abstracts, traced for mentor review | < $2 | mentors judge output quality |
| 1 | 100k abstracts + precision audit | ≈ $1k | audited precision acceptable |
| 2 | 1M abstracts, beta Databus release | ≈ $10k | community review of beta dataset |
| 3 | Full corpus, versioned release | ≈ $45–55k | — |

The ask is Phase 1 only. Phases 2 and 3 happen only if Phase 1's audited precision earns
them. (Phase 3 drops roughly 10× if we self host an open model instead of paying per token,
but that needs its own quality pilot first.)

Phase 0 is already done and in the appendix, the priest above (8/8 triples approved), a
village containment chain (`Medamarthy → Srikakulam_district → Andhra_Pradesh → India`,
4/4), and **one deliberately unflattering example**: a university abstract where the
pipeline produced `programCost "baccalaureate programs, graduate programs"` and a vacuous
`student "diverse student population"`.

I put the bad one in on purpose. Benchmark sentences come with a per-domain list of allowed
predicates; real Wikipedia abstracts do not, so predicate selection is looser in the open
domain. That is a real limitation, the precision audit in Phase 1 exists specifically to
measure it, and a proposal that shows only its best examples is not a proposal anyone
should fund.

### What's next: the final boss

Tommaso showed me a diagram that I have not stopped thinking about. The sentence:

> "Marie Antoinette's husband was killed in the war."

Two of the entities in that sentence are **never named**. Every extractor I know of does
one of two wrong things here: hallucinate a URI like `Marie_Antoinette's_husband`, or
quietly resolve it to `Louis_XVI` which is an *inference*, not an extraction. The
sentence did not say Louis XVI.

The correct representation is a **blank node**: there exists some x such that
`spouse(Marie_Antoinette, x)`, `gender(x, "male")`, `causeOfDeath(x, y)`,
`type(y, MilitaryConflict)`.

And here is why this problem fits this project specifically: **that graph is a SPARQL
query.** An anonymous node is not a constant to be guessed, it is a query to be answered
and I have a local triple store that answers queries in milliseconds. Ground it when the
answer is unique, keep it anonymous and provenance tagged when it is not. It is the
faithfulness principle I have been applying to facts all summer, extended to identity
itself.

That is the direction I want to take into the final weeks, and it is what a workshop paper
would actually be about, not +0.004.

### Where we stand

| Deliverable | Status |
| :--- | :--- |
| Benchmark result | **Macro F1 0.6317**, frozen at tag `v14-final`, reproducible, archived |
| Deployment | Public Docker image, verified end-to-end on two machines |
| Upstream | Merged into DBpedia's official repository |
| Scaling | Costed proposal with traced real world examples, on the table |

### Next Steps

Study my own proposal until I can defend every number in it, then pitch Phase 1. Then the
`wikidata_tekgen` split for generalisation, and the blank nodes. Lets go!
