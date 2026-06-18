---
layout: post
title: "GSoC '26 Week 1: Designing the Dual Approaches"
date: 2026-06-01
categories: gsoc dbpedia
---

With the Community Bonding period wrapped up, it was time to start writing actual code. The goal was clear: extract clean knowledge graph triples from messy natural language sentences. But the big question still remained: *how*?

We needed to design a solid approach to go from a messy sentence to clean, connected triples. Based on all the research and brainstorming I did during the community bonding period, I designed two distinct approaches to tackle the project goals.

## Approach A: The Traditional NLP Pipeline

**1) Entity Extraction (SpaCy):** Scan the input text and extract the main nouns from it using the SpaCy Python library. 
For example, given the input sentence: *"Inception was directed by Christopher Nolan"*, the output given by this first step of the pipeline would be `["Inception", "Christopher Nolan"]`. This gives us the potential subjects and objects of the triple.

**2) Candidate Retrieval:** Next, we take those plain text names and use the DBpedia API to get a list of all potential matches. As a result, we get the DBpedia URIs for all the matching candidates for those extracted nouns.

**3) Context Disambiguation (MiniLM):** Use a MiniLM model to eliminate the wrong candidates by reading the surrounding words in the sentence. The code encodes the full input sentence into a vector. It also encodes the summary descriptions (abstracts) of each candidate from step 2 into vectors. It then runs a cosine similarity matrix check between them. Now that we have the mathematically correct URIs for the nouns in the sentence, this step is officially termed "Context Disambiguation".

**4) Predicate Linking:** Once we have the URIs for the resources, it's time to find the relation between them—the ontology. We take the action phrase from the sentence and map it directly to an official property in the DBpedia dictionary. We use the official DBpedia `.owl` file that contains around 1,200 properties, and embed all 1,200 property labels into a fixed matrix using our local MiniLM model. The extracted text phrase is encoded into that same vector space, and we calculate the closest property in the `.owl` file.

**5) Final Sanity Check:** The last layer of the pipeline is a validation layer, running a final check to make sure we have generated a logically correct triple.

## Approach B: The LLM-Driven Pipeline

**1) Neural Extraction:** In this approach, we use a small LLM model (like REBEL or Qwen) to extract nouns directly from the input sentence. The model extracts raw text triples from the sentence and actively predicts the relationship between them.
For example, input sentence: *"Inception was directed by Christopher Nolan"*
Expected output text triple: `("Inception") -> ("directed by") -> ("Christopher Nolan")`

**2) Resource Mapping:** Next, we convert the raw extracted entities from the model into official DBpedia resources. Again, the DBpedia API is used for this. The API searches its indexed knowledge base and returns the most appropriate resource URIs for each entity.

**3) Predicate Alignment & Validation:** Predicate alignment serves as the final layer, just like in the previous approach, using the `.owl` file containing the 1,200 properties to ground the LLM's raw relation text into a strict ontology. After validation, we get our final triple. 

The plan for Approach B is to experiment with different LLMs to figure out which model performs the best and yields the highest baseline accuracy.

## Environment Setup & Next Steps

Once I had these two approaches mapped out and got the green light from my mentors to start implementing, I finally had the full picture in front of me—what to do and exactly how to do it. 

I set up my local development environment, installed the required dependencies, initialized the OpenRouter API keys provided by my mentors to load the models, and verified that everything was pinging correctly. With the setup completely dialed in, I was officially ready to start writing the code to put these two approaches to the test!
