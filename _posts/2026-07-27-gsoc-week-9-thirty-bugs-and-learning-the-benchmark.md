---
layout: post
title: "GSoC '26 Week 9: Thirty Bugs and Learning the Benchmark's Conventions"
date: 2026-07-27
categories: gsoc dbpedia
---

Last week the multi-triple refactor got us moving. This week was pure bug hunting, and it
was the most productive week of the project so far. Around thirty fixes, and one of them
taught me a lesson I am not going to forget.

### The bug that was loaded but unreachable

Let me start with this one because it is the most instructive thing that has happened to
me in this project.

I had written a literal normalizer — the function that formats dates, numbers, area codes
into the exact shape gold expects. I shipped it. Scores did not move. I checked the code
was loaded, `inspect.getsource()` said yes. I shipped a fix to it. Scores did not move
again. Checked again, still loaded. Three rounds of this.

The actual problem: the literal path emitted the object **angle-bracketed** like
`<sub> <pred> <"01325">`, but the wrapper regex that caught literals only matched the bare
form `"01325"`. So the normalizer was sitting there fully loaded and **completely
unreachable**. Every single check I ran returned True.

One character class in one regex was holding back three rounds of fixes.

**The moment it became reachable, `18_scientist` went from 0.4624 to 0.6129.**

The lesson, and I have written it on a sticky note: **verifying that code is LOADED is not
the same as verifying it PRODUCES the right output.** Test behaviour on a real sentence,
never the presence of a function.

### The bug that was deleting correct answers

This one was worse because it was silently destroying data.

Our SPARQL gate checks whether an entity actually exists in DBpedia before accepting a
triple. The code was:

```python
except Exception:
    return False       # <- a TIMEOUT was being treated as "entity does not exist"
```

A five second timeout on a public SPARQL endpoint is completely normal. And every single
time it happened, we deleted a correct triple.

I only caught it because the rejections looked absurd. `dbr:Manila` reported as not
existing. `dbr:Bank` reported as not existing. And on **every** rejection, both subject
AND object came back False simultaneously — which is the fingerprint of a failed query,
not a missing entity.

**44 correct triples destroyed in a single domain.**

The fix: three states instead of two. True (exists), False (confirmed absent), and
**None** (could not check). Timeout raised to 15s, one retry, and the gate now **fails
OPEN** on None. An unreachable server must not be able to delete a correct triple.

**`7_company` went 0.4309 → 0.5291.** Dead-URI rejections went 44 → 5.

### Learning conventions instead of hardcoding them

This is the theme of the week and the thing I am most careful about.

The benchmark has conventions that are not documented anywhere, you can only discover them
from the data. The temptation is to hardcode them. I did not want to do that, because
hardcoding what you observed in the test set is just fitting to the test set.

So every rule this week is **learned from the TRAIN split and validated across all 290
(domain, predicate) pairs** before shipping.

**a) Per-domain quoting conventions**

`9_astronaut` writes dates as `"1923-11-18"` — quoted. `18_scientist` writes `1776-02-18` —
unquoted. Same predicate, same range, opposite conventions.

Learned from train, it is 93% consistent within a (domain, predicate) pair.

Then a refinement that I like a lot. A strict majority vote got 277/290 pairs right but
failed on astronaut's `birthDate`, where train is a near tie — 9 quoted, 11 unquoted. But
quoting is a **convention**, not a coin flip. If a domain quotes something 45% of the time,
that is the convention showing through noise, not a tie.

So I used a **40% threshold** instead of a 50% majority. That gives 278/290, and the single
pair it changes across the entire benchmark is exactly the one that mattered.

**`9_astronaut` 0.4916 → 0.5723.**

**b) Middle initials**

Gold writes `Abraham_A._Ribicoff`. English DBpedia Lookup only returns the redirect
`Abraham_Ribicoff` — the canonical form with the initial exists only in the German and
French DBpedia!

So I wrote a deliberately narrow rule: restore the initial **only** when the resolved URI
is exactly the mention minus a single middle initial. It never expands a name, never
changes an entity. Tested: 3/3 target fixes, 6/6 dangerous cases untouched.

**`6_politician` 0.4623 → 0.5488.** Zeros went 30 → 14.

**c) Predicates gold quotes despite an entity range**

Some predicates are quoted literals in gold even though their ontology range says they are
entities — artist `background`, building `address`, astronaut `almaMater`, politician
`office`. The range-based detector missed all of them and the pipeline tried to entity-link
them.

Learned from train at an 80% threshold. **Recovers 52 test triples across 6 domains,
breaks 0.**

**d) Parenthetical units**

Gold keeps `6603633000.0 (kilometres)` with the unit inside. My normalizer was stripping
units. Correct for airport's inline `2194 feet`, completely wrong here.

**19-domain literal formatting went 85% → 93.1%.**

### What did NOT work

I want to be honest about this because I nearly shipped it.

The error analysis showed that 67% of the triples we never even attempted were geographic, 
sentences like "located in Perth, Australia" where gold wants BOTH a location triple and a
country triple, and we only emitted one.

So I added a rule to the Node 1 prompt telling it to decompose place hierarchies. On a
clean run of `4_building`:

* recall went UP: 0.3236 → 0.3366 ✓
* precision went DOWN more: 0.3725 → 0.3150 ✗
* **net F1 got worse: 0.3404 → 0.3220**

The model started inventing country triples that were not in the sentence. **Reverted.**

The lesson: "add the missing thing" is not automatically net positive. If adding it costs
precision at a worse rate than it gains recall, you have made the pipeline worse while
feeling like you improved it. The diagnostic was right about the problem and my fix was
still wrong.

### Where we stand

| Domain | F1 |
|---|---|
| 18_scientist | 0.6129 |
| 9_astronaut | 0.5723 |
| 6_politician | 0.5488 |
| 7_company | 0.5291 |
| 3_airport | 0.4623 |
| ... | ... |

The infrastructure also got a lot of attention this week — a per-sentence circuit breaker
with a 180s hard wall, and a runner with throttling, retries and cooldowns. Not glamorous,
but the public endpoints were timing out at 20 to 55% during long runs and I was throwing
away entire domain runs because of it.

### Next Steps

Next week: the model ablation my mentors asked for, and building the local Redis index to
finally get rid of the Lookup API. Lets go!
