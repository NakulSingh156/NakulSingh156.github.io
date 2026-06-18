---
layout: post
title: "GSoC '26 Week 2: Implementation, Hallucinations, and the Need for a Neuro-Symbolic Bridge"
date: 2026-06-08
categories: gsoc dbpedia
---

Now as I was ready with both of my approaches, I straight away started with implementing Approach B.
For that I started experimenting various LLM models, and compared their performance, by calculating the quality and accuracy of raw triples given by them. For testing those models and evaluating their performance I used 10 benchmark complex sentences.
As per my mentor's recommendation, I started experimenting with Small Language Models: qwen, gemma and llama.
I used the Openrouter API key for loading these models on the Jupyter server where I'll be doing all the work.
Now the results were very interesting, on one hand llama and qwen, especially llama was performing nicely. They were able to give me raw triples out of complex sentences, like ["Subject"]->["Predicate"]->["Object"].
But on the other hand gemma was hallucinating, it suffered from "Coreference Laziness"—if a sentence said, *"Christopher Nolan directed Inception. He also wrote it,"* the model would extract `("He") -> ("wrote") -> ("it")` instead of actually resolving the pronouns to the real entities. In other cases, it completely invented relational verbs that made sense in English, but had absolutely zero semantic value for a structured database.
To see exactly how bad this gets, look at this log from one of my early pipeline tests. The input sentence was: *"Quentin Tarantino wrote the screenplay for Pulp Fiction while Uma Thurman starred in it."*

![Gemma Hallucination Log](/hallucination.png)

**Coreference Laziness:** Gemma extracts the literal word `'it'` instead of resolving the pronoun to *Pulp Fiction*.

## Upgrading the Engine: Qwen vs. Llama 3

After the catastrophic failures with Gemma, I realized that the pipeline required a more robust foundational LLM for Step 1. I decided to benchmark two heavyweights available via OpenRouter: `qwen/qwen3-8b` and `meta-llama/llama-3-70b-instruct`. 

First, I tested Qwen. The results were definitely more accurate. Look at the pipeline output below for the exact same input style that Gemma failed on:

![Qwen Extraction Success](/qwen-success.png)

Qwen flawlessly bypassed the coreference trap. It cleanly extracted `'Christopher Nolan'` as the subject, `'directed'` as the raw predicate, and `'Inception'` as the object. From there, my Neuro Symbolic math layer took over and perfectly mapped them to the official DBpedia URIs.

### The Final Verdict: Why Llama 3 Won

The deciding factor was **API Latency and Throughput**. 
For a knowledge graph extraction pipeline to be viable in the real world (especially when batch-processing thousands of sentences from any benchmark dataset), speed is critical. While both models demonstrated near-identical accuracy in zero-shot/few-shot atomic extraction, my OpenRouter telemetry showed that Llama 3 consistently delivered a lower Time-to-First-Token (TTFT) and significantly faster overall generation speeds. 

![Latency and Triple Tracker](/latency-matrix.png)
*System Performance Matrix tracking Avg Latency and Total Triples Extracted.*

The data tells the whole story:
1. **Gemma (2.279s | 12 Triples):** It was the fastest, but recklessly so. It over-fragmented the sentences, spitting out 12 noisy and broken triples.
2. **Qwen (4.095s | 10 Triples):** Highly accurate, yielding exactly the 10 correct relational triples we expected. However, it was the slowest of the bunch.
3. **Llama 3.3 70B (3.990s | 10 Triples):** The ultimate sweet spot. It matched Qwen's perfect semantic accuracy but executed faster, edging it out in latency.

Llama 3 gave me the exact same semantic accuracy as Qwen, but it shaved crucial milliseconds off every single API call, allowing the Python math layers to execute faster. We had our winning model.

With this the step 1 of the pipeline is complete, we are able to generate raw triples from the sentence using Llama model. Now the next step of the pipeline is fetching the 15 most relevant URIs for the resouces in the raw triple using the DBpedia API. We take the context of the sentence in a vector space, now our MiniLM model calculates a cosine similarity score with the abstract of each of the 15 URIs, and then all those candidates are ranked on the basis of that similarity score.

We faced similar hallucinations in this layer as well. The code executed perfectly and did the math correctly, but the actual output was wrong because of what I call **"Context Drowning."**

Pure vector models get overwhelmed when candidate abstracts contain the exact high context proper nouns from the input sentence. Look at this visual proof from my research log during the Pulp Fiction test:

![Mia Wallace Context Drowning Visual Proof](/mia_wallace_fail.jpg)

**The Failure Breakdown:**
The pronoun "it" in the sentence text was supposed to resolve to the movie **Pulp Fiction**. However, because our input sentence was packed with keywords like "Uma Thurman" and "Quentin Tarantino", the DBpedia lookup candidates pulled both `dbr:Pulp_Fiction` and `dbr:Mia_Wallace`.

MiniLM calculated the highest similarity score for the character **Mia Wallace**. Why? Because her abstract explicitly says: *"fictional character portrayed by **Uma Thurman** in **Quentin Tarantino's film Pulp Fiction**..."*

The character description contained almost every single proper noun from our input sentence. The model drowned in the context and ranked `dbr:Mia_Wallace` as the top match instead of the actual movie. The pipeline confidently claimed that Uma Thurman starred in the *character* Mia Wallace, rather than starring in the *movie* Pulp Fiction!

## The Solution: The Neuro-Symbolic Bridge

This failure proved that pure semantic vector similarity isn't enough on its own. We need a smarter and more dynamic approach to calculate the similarity score for each of the URIs, and then rank them based on this smarter approach. This is exactly why we need a hybrid, **Neuro-Symbolic** approach:

**Semantic Self-Learning Entity Disambiguation:** We need to implement a lexical anchor. The pipeline needs to dynamically balance the context vector (meaning) with strict string matching (Edit Distance) on the entity name to prevent context drowning.

I now had the blueprint for the solution. Next week, I'll be deep in the trenches writing the Python code to implement this exact Semantic-Lexical blend.
