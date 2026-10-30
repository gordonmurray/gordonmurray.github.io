---
layout: post
title: "Trying to improve BM25 with AI agents"
date: 2026-10-30 09:00 +0000
categories: data
tags: ["paperclip", "bm25", "jev", "firn", "codex", "search", "benchmarks"]
---

I wanted to see how far I could get with autonomous AI agents working on a technical research problem. I didn't want to ask Codex to implement something. I wanted to give a group of agents a problem, let them investigate it, run experiments, learn from the results, and hopefully come up with something worth keeping.

I'd recently been experimenting with Paperclip, which provides a way to organise coding agents into projects, tasks and roles. My [earlier experiment with the Rust bbtree library]({% post_url 2026-10-26-trying-paperclip-with-a-bbtree-experiment %}) had worked, but keeping the agents progressing required more intervention than I'd expected. This time I wanted to give them something bigger.

I settled on trying to improve BM25, the ranking algorithm used by search engines such as Elasticsearch and OpenSearch. More specifically, I wanted to see whether AI could help discover a mathematical formula that approached the relevance quality of an AI reranker called Jev, without needing to run an AI model for every search.

It sounded like a good challenge for autonomous research. The agents could propose formulas, implement them, run benchmarks, examine failures and keep trying. That was the plan anyway.

## Why BM25 and Jev?

I've been working on [Firn](https://github.com/gordonmurray/firnflow), my open-source vector and full-text search engine backed by object storage. In September, I'd [experimented with Jev]({% post_url 2026-09-20-does-jev-improve-firn-search %}), a model from TypeSafe designed to make fast, structured decisions. Rather than generating text, it can evaluate a question about some supplied content and return a score.

For search, that means taking documents returned by BM25 or another retrieval system, asking Jev whether each document is relevant to the query, and using the resulting scores to reorder them. Jev had performed well in my earlier Firn experiments.

BM25 works differently. It ranks documents using statistics about their words, mostly how often a term appears, how rare it is across the collection and how long the document is. It's been around for decades and is still a very good baseline.

I wanted to find out whether we could discover a relatively simple formula that produced better rankings than BM25, using Jev as a teacher. If that worked, we could get some of the relevance improvements of AI reranking without the inference cost or latency.

## Giving Paperclip the problem

I set up a research project with several roles, including a research lead, ranking researcher, storage researcher and an evaluation agent responsible for checking the results. The agents used Codex to do their work, and Paperclip coordinated tasks and reviews.

When I say "we", I mean the agents and me. I set the direction and stepped in when needed; the agents wrote code and ran experiments.

I also wanted to fix the reliability problems I'd run into last time. Instead of relying on an agent session to keep an experiment running, I had Codex build a separate experiment runner. It used SQLite to track jobs, kept results and configurations, and could resume work after an interruption.

That meant the agents could investigate and write code, while experiments ran on their own with fixed inputs, resource limits and recorded outputs. I also added monitoring to detect stalled work.

The project did complete its first campaign, but plenty of questions were still open. Some experiments had failed before producing usable measurements, and their tasks were marked complete anyway because the failure reports had been written. A finished task and a research question that had moved forward were two different things.

## A document moved from rank 66 to rank 1

The first result worth digging into came from SciFact, a public benchmark containing scientific claims and abstracts. We started with 30 development queries and compared BM25 with Jev reranking the same 100 candidate documents.

One query was about T-cell activation, specifically the requirement for signals from the T-cell receptor alongside secondary signals from costimulatory receptors. BM25 placed a judged-relevant paper about CD28 signalling at rank 66. Jev put it first.

The paper described CD28 providing a critical second signal alongside TCR ligation. Another highly ranked document discussed similar entities but included independent signalling. Both documents contained relevant terminology. The difference seemed to be in how those terms related to one another.

I couldn't know what Jev had recognised internally, but it looked like something that ordinary word statistics would struggle with.

Across the 30 queries, the results were:

| Method | nDCG@10 |
|---|---:|
| BM25 | 0.7535 |
| Jev | 0.8502 |

nDCG@10 measures how well the first ten results are ordered against human relevance judgments. Higher is better.

That was a big improvement, although 30 queries weren't enough to show that Jev would be better everywhere. Still, it was exactly the kind of result I wanted the agents to investigate. Could we figure out what Jev was doing differently and reproduce some of it mathematically?

## Finding a better formula was harder than expected

The agents started by generating and evaluating hundreds of candidate mathematical expressions. One looked promising on the development queries, improving nDCG@10 from 0.7535 to 0.8002. Then we tested it on 300 separate SciFact queries. BM25 scored 0.6605 and the new formula scored 0.6163, so it was worse.

Generating hundreds of formulas makes it easy to find one that does well on a small set of queries. That doesn't mean the formula has learned anything that carries over to new queries.

There was another problem. The agents had optimised the formulas against the benchmark's human relevance judgments, not Jev's decisions, which wasn't the experiment I'd asked for.

I went back to Codex and asked it to implement the Jev comparison directly. This time we used 126,436 pairwise ranking preferences derived from cached Jev scores and trained a linear scoring function using eight inexpensive features.

The formula was essentially a weighted sum of the eight features for each query and document:

`score(query, document) = w1 × feature1 + w2 × feature2 + ... + w8 × feature8`

Each feature represented something measurable, such as BM25 score, query-term coverage, term frequency, document length or word proximity. The model learned weights for those features from Jev's ranking preferences.

This produced a modest improvement over BM25 on the original development queries. We froze the formula and tested it on 1,000 previously unused SciDocs queries. It improved nDCG@10 by about 2%, but the confidence interval included zero, so we couldn't say the improvement was real.

Worse, the learned formula still ranked the CD28 document 67th. We had trained on Jev's preferences, but hadn't reproduced the one decision that got me interested in Jev in the first place.

## What exactly were we teaching the model?

The training data had another problem. Of the 126,436 pairwise preferences, 97.65% compared documents without a positive relevance judgment.

Those documents weren't necessarily irrelevant, since many were unjudged. But it meant most of our training objective was about reproducing Jev's ordering of documents the benchmark had no positive relevance evidence for.

We tried changing the objective to focus more on Jev's highest-ranked results. That nudged the average benchmark score up a tiny amount, but it still didn't recover the important promotions. The CD28 document dropped further, to rank 84.

Eventually, I asked Codex to check whether any combination of our eight features could reproduce the ranking decisions we wanted. For three important documents, it found separate sets of weights that could place each document first. But it also showed that no single shared set of weights could satisfy all the selected Jev preferences across those examples. Nine conflicting ranking requirements were enough to prove this using exact arithmetic.

So the features could tell individual examples apart. One global linear formula just couldn't satisfy all those requirements at once. That didn't rule out a useful compromise or a nonlinear formula, but it gave us a reason to stop searching for more linear weights.

We'd also been measuring how many useful Jev/BM25 disagreements a formula could repair. Reversing BM25's scores satisfied all 98 selected disagreement pairs. That makes sense, since we'd picked those pairs because Jev disagreed with BM25, but the resulting rankings were terrible everywhere else. Our metric had rewarded exactly the wrong behaviour.

## A side experiment with S3 costs

Since Firn uses object storage, I also gave the agents a secondary problem. Could a search engine avoid fetching some postings blocks from S3 without hurting the results much?

Postings are the index entries that identify which documents contain a term. Fetching fewer blocks could mean fewer S3 GET requests, less transferred data and potentially lower latency.

The early results looked good. One approach cut simulated GET requests by about 22%, but it also damaged relevance badly.

Skipping blocks turned out to cause two different problems. Some relevant documents were never discovered, and documents that were discovered could have incomplete BM25 scores because some term contributions were missing.

We experimented with selectively fetching additional blocks. On NFCorpus, fetching one additional block recovered a surprisingly large share of the available ranking improvement, and I thought we might finally have something useful.

Then we froze the strategy and tested it on FiQA, a different dataset, and it didn't hold up. One additional block recovered only about 36% of the available gain, compared with about 88% on NFCorpus, and the number of missing blocks per query was very different between the two datasets.

We closed the storage research without integrating anything into Firn. The request savings were simulated using a model of postings-block access, not measured against real S3 traffic. Even if relevance had held up, we'd still have needed a realistic implementation and network benchmark.

## Was Jev actually as good as we assumed?

Up to this point, we'd been trying to copy Jev. But the original evidence came from a small set of SciFact queries, and I was getting uncomfortable treating Jev's decisions as the correct answer. So we tested it more thoroughly.

On separate NFCorpus and FiQA evaluations, Jev improved relevance against human judgments, even after we strengthened the conventional lexical baselines.

| Dataset | Improved lexical baseline | Jev |
|---|---:|---:|
| NFCorpus | 0.3119 | 0.3728 |
| FiQA | 0.2316 | 0.4017 |

Improving conventional lexical retrieval helped a fair bit too. Stemming, BM25 tuning and pseudo-relevance feedback improved the baseline, particularly on NFCorpus, but most of Jev's advantage remained.

Some of the failures were telling. For a query comparing antibiotics in white and dark meat, a lexical search ranked a chocolate study first. It was giving too much weight to certain matching words and feedback terms, rather than the main subject of the query. For another question, Jev recognised that equities and stocks were relevant alternatives.

We tried inexpensive query-term weighting and lexical adjustments to address these problems, but the improvements were negligible. By now we'd tried a lot of variations of the same general idea without getting particularly close to Jev.

## Giving the research more resources

I was getting frustrated with how quickly each experiment finished. I'd given the project access to a desktop with an Intel i9-12900KF, 31 GB of RAM and an RTX 3080 GPU, yet several experiments were using one CPU core, training a model in a second or two, reporting a small improvement and recommending that we stop. That wasn't what I'd pictured when I started experimenting with autonomous research.

So I changed the objective. Instead of insisting on a BM25-like mathematical formula, I asked the agents to investigate how cheaply we could reproduce Jev's relevance quality. That let them explore neural models, learned representations, different training methods and model compression.

The larger campaign ran for about 2 hours and 38 minutes and used 1.17 million agent tokens. It used GPU inference, evaluated several model configurations, and every result was checked by a separate verification step. The experiment runner also recovered from a checkpoint after being interrupted.

But the final results had another surprise in them.

## Jev wasn't better everywhere

The final campaign evaluated two more retrieval datasets, Android and Touché. Android involves finding relevant material for Android-related questions, while Touché is an argument-retrieval benchmark, a rather different task.

The selected model was a MiniLM reranker trained using human relevance judgments from 1,937 queries. The final results were:

| Method | Android | Touché |
|---|---:|---:|
| Lexical baseline | 0.3792 | **0.3361** |
| Trained MiniLM | 0.4411 | 0.2892 |
| Jev | **0.4696** | 0.2845 |

The trained model improved Android a lot, but it did worse than lexical search on Touché. So did Jev.

We'd spent a lot of effort trying to reproduce the decisions of a model we'd assumed was generally better at ranking, and it wasn't. At least, not with the question we were asking it and the datasets we'd evaluated.

A separate experiment on Quora pointed the same way. Changing Jev's instructions to match the duplicate-question task improved nDCG@10 from 0.688 to 0.839 on a small 20-query diagnostic. The model hadn't changed. The question had.

So some of what we'd been measuring might have been how well our instructions matched the retrieval task, rather than a universal difference in ranking ability. We hadn't tested task-specific Jev prompts on every final dataset, so I couldn't conclude that Jev was unsuited to argument retrieval. But I couldn't keep assuming it was the better answer either.

A pretrained MiniLM model also did very well on Android before any of our additional training, so more training data and compute didn't automatically produce a better model. The original small LightGBM model held up surprisingly well in the final comparisons too.

By the end, we had more models, more experiments and a lot more evidence, but no general replacement for BM25 or Jev.

## Keeping the agents working was a separate problem

Looking back, I learned almost as much about the research process as I did about search.

Paperclip was useful for assigning work, following tasks and getting agents to review one another's output. But a running orchestrator didn't mean progress was being made. At one point, a monitor kept running while a review was blocked. The process was alive, but the project wasn't moving.

I kept going back to Codex to find out why things had stopped, ask it to recover work, or point out that a failed experiment wasn't the same as completing the research.

The durable experiment runner helped. Jobs kept their inputs, checkpoints and outputs, so an interrupted run didn't mean losing the work. In later tests, a worker could be killed and restarted without corrupting the results, which was a real improvement over relying entirely on agent sessions. I never saw it recover from every kind of Paperclip failure on its own, though.

Giving agents a budget and permission to continue wasn't enough either. They were very good at completing a defined experiment, producing a report and recommending a stopping point. They were much worse at deciding whether the wider research question had been explored properly, so I kept having to change the instructions, widen the scope or push back on a conclusion.

Independent verification was essential. Several promising findings disappeared when tested on different datasets. Other experiments had implementation or data-validation problems that needed fixing before their results meant anything. Green task statuses told me very little. I had to look at the measurements.

## Where I ended up

I started this wanting to see whether autonomous AI agents could discover a better ranking algorithm. We didn't manage that.

The Jev API spending across the project came to about $4.72. That excludes the cost of my Codex and other AI subscriptions, desktop electricity and my own time. The final large campaign alone used over a million agent tokens. The GPU and extra compute didn't produce the breakthrough I'd hoped for, but the experiments did change some of my assumptions about evaluating AI models and running autonomous research.

For another project, I'd do a few things differently. I'd set up several genuinely different evaluation workloads before choosing a model to copy. I'd keep a set of untouched tests until the end and pay more attention to the number of independent queries rather than the number of document pairs. I'd separate experimental jobs from the agents managing them, keep the independent verification, and make the recovery rules explicit from the start.

I'd also give the agents clear authority to keep investigating a problem, while making sure they couldn't mistake completing a task for answering the research question. And I'd think harder about whether I was asking the right question in the first place. We spent a lot of time asking how to reproduce Jev's rankings, and reproducing Jev turned out to be a different problem from improving search.

I'm leaving Firn unchanged and closing the research project for now. I may come back to autonomous research with a larger project in the future. If I do, this is probably the post I'll want to read first.
