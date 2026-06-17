---
layout: post
title: "GSoC '26: Community Bonding Period"
date: 2026-05-24
categories: gsoc dbpedia
---

It was the night of 30th April 2026, around 11 pm. I was patiently waiting for the official list of GSoC contributors to be released, which was supposed to come at 11:30 pm. I just kept refreshing my inbox. At around 11:37 pm I received the mail: "Congratulations you have been accepted as a contributor for GSoC 2026!" You know, I was constantly visualizing how I would celebrate if I get selected, and after I did get selected, my reaction was surprisingly very calm. Like yeah, I've worked for it.

## Kicking Off the Community Bonding Period

So we kicked off our GSoC period with the community bonding period. I got a chance to meet my fellow contributors for DBpedia, and interact with my mentors for the project. To discuss the project goals and to talk about the project timeline, setting up the weekly goals till the end of the GSoC period. I presented the project timeline I put in my proposal, which talked about scaling my Neuro Symbolic Pipeline on the whole 50 GB DBpedia dump. During January - March 2026, when I was contributing to DBpedia's Neural project, while fetching any data from the DBpedia data, there used to be hallucinations and errors. I engineered a validation layer to validate the data extracted with the official DBpedia set of rules. And I also made a BFS layer to find the connection between two DBpedia relations, like if I want to find out the relation between two entities in the knowledge graphs. The layer would implement BFS algorithm to actually iterate through the complex knowledge graph to find the relation between the two entities. My plan for the coding period was to scale this Neuro Symbolic Prototype, and implement this on the whole 50 GB DBpedia dump and set it up on a local docker based redis architecture.

## The Mentor Connect & Project Course Correction

In one of my weekly connects with my project mentors, Tommaso Soru, Ara Yeroyan and Nandana, they gave me a very clear perspective of the project, the project goals and what's expected from me. Instead of trying to scale my small prototype on the whole 50 GB DBpedia dump, and setting it all up locally in a docker based redis architecture, which requires a lot of computation and resources!! I should focus on extracting the correct relations from the data from a messy sentence, which is the core goal of the Neural Extraction Project!! 

We have a messy or a complex sentence as input, and as output we should be able to generate triples from that sentence and find the relations between them. 
For example, we have an input sentence like "Inception was directed by Christopher Nolan".
As output we need to be able to generate a triple like this: `dbr:Inception -> dbo:director -> dbr:Christopher_Nolan`.
Correctly identifying the subject, the object and the relation between them, the ontology.

## Researching Paths to Implementation

That is where I got total clarity of what I have to do in my project. And I had a goal in front of my eyes. So I spent my time researching about building an approach of how to implement this. I saw the last 2-3 years project repos and saw what those contributors did, and more importantly how they did it, their approach and the key points from it which I can integrate in mine and then build upon that.

I also referred to some research papers on Neuro Symbolic AI, to get an idea of how things work in this space and what work has been done till now. This gave me a few points to implement in my project and also some things which I can build upon. All these learnings helped me to design the approaches for implementing my projects in the future.
