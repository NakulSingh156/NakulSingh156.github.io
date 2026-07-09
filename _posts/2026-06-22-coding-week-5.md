---
layout: post
title: "GSoC '26 Week 5: The 4-Node Architecture and the Diagnostic Judge"
date: 2026-06-29
categories: gsoc dbpedia
---

Last week, I ended the blog by mentioning the need for a validation layer. Even with optimized math blends and soft penalties, statistical anomalies still slip through. We needed a definitive guardrail.

This week, we completely overhauled the system into a **4-Node Autonomous Neuro-Symbolic Architecture**. 

### Why not "just use an LLM"?

A common critique I faced while building this was: *If the math gets it wrong, why not just pass it to an LLM to fix the output?*

The answer is simple: LLMs are stochastic. If you ask an LLM to "fix" a mathematical ranking error, it just guesses the weights. We wanted a deterministic approach. By replacing LLM guesswork with a deterministic **Algebraic Margin Solver**, our architecture achieves:

* **Zero-Cost Latency:** Math solves in nanoseconds; LLM calls take seconds and cost tokens.
* **Determinism:** Given the same error, the algebraic solver computes the exact same corrective weights every single time.
* **Explainability:** Every ranking shift is mathematically provable, eliminating the "black box" effect.
* **Parametric Failsafe:** The LLM is reserved *only* as the ultimate auditor and final failsafe, maximizing efficiency.

---

### The 4-Node Breakdown

Here is how the pipeline flows now:

1. **Node 1: Strict Atomic Extraction (The Neural Filter)**
   We use Llama 3.3 70B to parse the raw sentence and extract the minimal Subject, Predicate, and Object strings, stripping away noisy clauses.
2. **Node 2: Targeted Retrieval (The Symbolic Fetcher)**
   We ping the DBpedia Lookup API to fetch the top 15 candidates for both entities.
3. **Node 3: Independent Resolution Math Engine**
   We score the entities using our multi-dimensional vector/lexical blends (80/20 for entities, 50/50 for predicates) and apply the ontological penalties I discussed last week.
4. **Node 4: The Diagnostic Judge & Algebraic Solver (The Healer)**
   This is where the magic happens.

---

### Node 4: The Healer

Node 4 acts as the diagnostic meta-reasoner. The LLM looks at the Rank 1 triple produced by Node 3 and audits it against the original sentence. 

If it's right, it approves it. But if the LLM detects that Rank 1 is an "Imposter", it doesn't just hallucinate a fix. It passes the URIs to my custom Python **Bidirectional Algebraic Margin Solver**. 

The solver dynamically scans the raw sub-scores stored by Node 3 and executes a linear scan to find the exact mathematical boundary where the Target outscores the Imposter. It updates the graph's weights (e.g., shifting from `0.75/0.25` to `0.00/1.00`) and triggers a loop back to Node 3 to self-correct the ranking!

---

### The Apple Oversight

While this loop is powerful, it still has blind spots if the LLM Judge isn't strict enough. Let's look at this test case:

> **Input:** *"Apple released the new iPhone."*
> 
> **Node 1 Extraction:** `[Apple] -> [released] -> [iPhone]`
> **Node 3 Resolution:** `<Apple_Inc.>` -> `<releaseLocation>` -> `<IPhone>`
> 
> **Node 4 Verdict:** APPROVED. 
> *Diagnostic Feedback: The Rank 1 candidate perfectly matches the input sentence with Apple Inc. as the subject and iPhone as the object.*

See the problem? The math engine picked `dbo:releaseLocation` for the predicate "released". Node 4 looked at it, saw that Apple and iPhone were correct, and lazily hit "APPROVED". But the iPhone is obviously not a location! Node 4 hallucinated a pass because its prompt wasn't strict enough on domain/range logical alignment.

---

### Next Steps: LangGraph and Strict Prompting

To fix this and handle these cyclic correction loops elegantly, my primary task for next week is to implement **LangGraph**. This will make the entire system natively agentic and handle the cyclic routing between Node 3 and Node 4 seamlessly.

I will also be rewriting the Node 4 prompt to be much stricter. I plan to add few-shot examples directly into the system prompt—feeding it this exact "Apple / releaseLocation" example—so it learns to never hallucinate or approve poorly matched predicates again. Stay tuned!
