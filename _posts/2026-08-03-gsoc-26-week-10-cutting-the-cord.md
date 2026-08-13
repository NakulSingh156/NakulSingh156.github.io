---
layout: post
title: "GSoC '26 Week 10: Cutting the Cord — Building a Local Index and a Local DBpedia"
date: 2026-08-03
categories: gsoc dbpedia
---

Last week was thirty bug fixes. This week was one idea: stop asking the internet
questions we can answer ourselves.

Two web services sat in the middle of my pipeline. DBpedia Lookup, which turns a mention
like "Mermaid" into candidate entities. And the public SPARQL endpoint at dbpedia.org,
which my fact gate queries to check whether a triple is real. Both were the slowest parts
of the system, both rate-limited me during long runs, and as the error analysis kept
telling me, one of them was the single biggest source of lost F1 in the whole project.

The diagnosis I had been avoiding
From my error binning: 105 of 215 errors (49%) were subject or object mis-resolution.
The clearest one, from 4_building:

GOLD:  108_St_Georges_Terrace | completionDate | 1988
OURS:  Manchester_City_F.C.   | completionDate | 1988

For weeks I read this as a ranking problem and tried to fix my scoring. It is not a
ranking problem. DBpedia Lookup returns almost nothing useful for an obscure Perth office
block, so a high connectivity football club wins on score. My Node 3 scoring was sound.
It simply never saw the correct candidate.

You cannot rank your way out of a candidate list that does not contain the answer.

The local surface form index
So I built the candidate list myself, offline, from the DBpedia dumps.

The idea is a phone book of every way any Wikipedia page has ever been referred to in
text, its title, its redirects, and the anchor texts of links pointing at it. Three
Databus artifacts (labels, redirects, disambiguations, lang=en) resolved live
against the Databus SPARQL endpoint so the build always picks up the latest release,
streamed through a builder, and loaded into Redis.

16.2 million mention entries. Lookups take about a millisecond.

Ranking inside the index is tier dominant: an exact page title beats a redirect, which
beats a disambiguation entry, with popularity as the tiebreak.

The thing I did not expect: literals became deterministic
This is my favourite result of the week and it was free.

I had been treating "should this object be an entity URI or a quoted literal?" as a
statistical problem, learn the convention per predicate from train, which is what Week 9
was about. Then I looked at the actual values:

TRAIN:  campus -> Dijon                        "Dijon" IS a DBpedia resource
TEST:   campus -> "In Soldevanahalli, Acharya  that string is NOT a
        Dr. Sarvapalli Radhakrishnan Road,     DBpedia resource
        Hessarghatta Main Road, Bangalore"

There is no inconsistency. The rule is: an entity typed object is written as a URI when
its surface form exists in DBpedia, and as a quoted literal when it does not. One rule,
zero exceptions.

I could never implement that rule against the Lookup API, because the API fuzzy-matches
something for almost any string, it never cleanly says "this is not an entity." The
local index does. An empty result is now an exact signal that flows straight through to
the emitter as the literal path.

A statistical approximation became a deterministic rule because I changed the data source.

The local knowledge base
Then the fact gate. My tiered SPARQL gate asks questions like "does the triple
(Bacon_sandwich, country, United_Kingdom) exist?" and "are these two entities connected
by anything at all?" Several queries per triple, every one a round trip to a public
endpoint that was timing out on 20–55% of requests during long runs.

So I bulk-loaded DBpedia into a local oxigraph store. Nine dumps, smallest first with a
delete after load step and a disk watchdog, because my server has 75GB and the load peaks
higher than you would guess:

dbpedia_text2sparql_ontology.nt   instance_types_en.ttl
instance_types_transitive_en.ttl  persondata_en.ttl
labels_en.ttl                     short_abstracts_en.ttl
mappingbased_literals_en.ttl      mappingbased_objects_en.ttl
infobox_properties_en.ttl

18GB at load, 15GB after compaction, served read-only on localhost.

The ablations
My mentors asked for these and they turned out to be the most convincing artifacts of the
whole week. Same code, same sentences, one component swapped.

Index off vs index on, 4_building: 0.450 → 0.647.

Remote gate vs local gate, 4_building: 0.524 → 0.561. Across the benchmark the local
gate is worth about +0.037 macro on identical inputs. That surprised me, I expected
the local gate to buy speed and nothing else. It buys accuracy too, because a timeout that
the gate cannot distinguish from "this entity does not exist" is a correct triple deleted.
Week 9 taught me to fail open on that; this week I removed most of the failures entirely.

And the speed: ~120s per sentence at the worst against dbpedia.org, ~26s locally. A
full domain that used to be an overnight gamble now finishes inside an hour.

What did NOT work
Tier 1.6, my range membership direction check. It looks at whether the emitted object sits
outside the predicate's declared range while the subject sits inside it, and swaps them if
so. Designed against the remote endpoint, where type data was rarely reachable, so it
fired maybe eight times a run and was net positive.

Against the local store, type data is always available, and the heuristic over-fires. I
labelled every swap it made on a clean 4_building run: roughly 7 harmful, 2 neutral,
0 helpful. Every genuinely good swap came from the inverted-exact ASK, which is a
different tier and stays on.

I did not delete it. I put it behind a flag defaulting to False, with the measurement
written next to the flag, because "this heuristic is right on sparse data and wrong on
dense data" is worth keeping in the code as knowledge rather than as a commit message.

The lesson: a heuristic tuned for a scarce signal can invert when the signal becomes
abundant. Changing your infrastructure invalidates the tuning you did against the old
infrastructure. I had not thought about that before this week.

Also this week
Switched the extraction model to GPT-5.6 Luna, dramatically cheaper per token,
which matters because my pipeline makes 8–12 LLM calls per sentence.
Object-side gate misses now demote to a quoted literal instead of deleting the whole
triple. Found on a sentence where gold wanted the literal "Government of Addis Ababa"
and the gate was destroying the entire currentTenants fact.
A reproducibility check I will keep repeating: 10_comicscharacter run on my Mac and on
the server scored 0.4111 vs 0.4120, Δ0.001.
Where we stand
Everything except the LLM call now runs on my own machine. No Lookup API, no dbpedia.org,
no rate limits, no outages.

Next Steps
The full 19-domain sweep. Every fix from Weeks 9 and 10 is in, the infrastructure is local
and fast enough to actually finish, and I finally get a number for the whole benchmark
instead of a handful of domains. My realistic expectation is 0.55, optimistic 0.57–0.58.
Lets go!
