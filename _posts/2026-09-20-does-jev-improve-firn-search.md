---
layout: post
title: "Does Jev improve Firn search? Measuring relevance, latency and cost"
date: 2026-09-20 09:00 +0100
categories: data
tags: ["firn", "jev", "search", "reranking", "open-source"]
---

Jev was mentioned all over X this week, so I wanted to try it out for myself. I joined the wait list and got access the next day.

Jev is TypeSafe's first System One model, a class of model they describe as built to make fast, structured decisions that software can use directly. It does not write text. You give it some text and a yes/no question, and it returns a probability.

Firn is my open source vector and full-text search engine, which you run yourself on top of object storage such as S3.

The question I wanted to answer is: does Jev put more useful results near the top of Firn's list, and what does that cost in latency and money?

## The baseline

The baseline is Firn alone (version 0.9.6), with no reranker: hybrid search (vectors from the BGE-small embedding model plus BM25 text search) returning 50 results per claim.

I test it on BEIR SciFact, a public set of 5,183 scientific abstracts and 300 test claims. Each claim is a one-sentence scientific statement, and the dataset's annotators marked which abstracts are relevant to it, usually just one.

I score the results with nDCG@10, which looks only at the top 10. A relevant abstract earns more credit the higher it sits, from full credit at rank 1 to under a third at rank 10, and nothing outside the top 10. The scale runs from 0 to 1.

Firn alone scores 0.727. Vector search alone scores 0.713 and BM25 alone scores 0.689. In plainer terms, a relevant abstract is the first result for 61% of claims and in the top 5 for 80%. Every figure below is compared against that 0.727.

## Reranking with Jev worked

I used Jev as a reranker, a second and slower step that re-sorts a search engine's top results. For each claim, Firn returns its top 50 results, which I call the candidates. I send Jev the claim and one candidate abstract with a yes/no question: does this abstract provide evidence that helps assess whether the claim is true, either by supporting it or by contradicting it? Jev returns a probability, and the candidates are re-sorted by it. I wrote the question using 100 training claims and froze it before scoring the 300 test claims.

nDCG@10 rose from 0.727 to 0.800, a gain of 0.073 (95% interval 0.046 to 0.100). In practice, a relevant abstract was the first result for 70% of claims instead of 61%, and in the top 5 for 87% instead of 80%. Of the 300 claims, 82 got better, 28 got worse and 190 were unchanged.

Claim 623 says people with low vitamin D levels are more likely to get multiple sclerosis. The paper the annotators marked as relevant is about sun exposure and multiple sclerosis. Firn had it at rank 13, outside the top 10, so it would not have made the first page and contributed nothing to the score. After reranking it was at rank 1.

Not every change was an improvement. Claim 793 says mitochondria play no part in cell death. The relevant abstract is a general review of mitochondria, and it fell from rank 2 to rank 13. From its title that looks like a fair demotion, but the review says in one sentence that mitochondria do take part in cell death, which bears directly on the claim, and Jev rated it 0.94. Twelve other candidates were rated higher, between 0.95 and 0.97, so a rating of 0.94 was not enough to stay near the top.

Most of the gain came from the first candidates. Reranking only the top 10 gave +0.048, the top 25 gave +0.070 and all 50 gave +0.073, so going from 25 to 50 added little for twice the calls. A search costs about $0.0008 in Jev calls at 25 candidates and $0.0016 at 50.

Firn alone took a median of about 40 milliseconds per search in this setup. With Jev reranking 50 candidates, the median became roughly 517 milliseconds. I sent the 50 scoring requests concurrently, so each search waited for the slowest response.

That is one dataset, so I would not assume the same gain on other data. But it did what I hoped.

Even so, I decided that I did not want to add a paid dependency to open source Firn. The price is small, about $0.0016 a search, so cost alone would not have stopped me. Firn is meant to run on your own infrastructure, and a reranker that needs an account with an outside company changes that: every user would need the account and live with its request limits.

## classifier.dev

I briefly tried classifier.dev, a free service that sits on top of Jev. You send it a batch of texts and a set of labels and it rates each text against them, so a whole search takes one request instead of one per candidate.

Its nDCG@10 score was almost identical to calling Jev directly: 0.796 against 0.797 when reranking the top 25.

But the ratings were squashed. 78% of candidates were rated at or below 0.01, against 5% when calling Jev directly, so there were only 6.3 distinct values per claim to sort on, against 10.4. Candidates with the same rating stay in Firn's original order, so a squashed scale leaves more of the final order to Firn.

There were limits I didn't like either. The free tier allows 20,000 classifications a day per IP address, about 800 searches a day at 25 candidates. There is no model parameter, so I could not pin the Jev version. So I passed.

## Open source rerankers

So I looked at open source rerankers instead. These are small models that read a claim and one abstract together and output a relevance score. They run on your own hardware, so there is no per-search fee and no outside service to depend on. Firn is Apache-2.0, so I only considered permissive licences. The jina-reranker v2 and v3 models are CC-BY-NC and were out.

I tried five, each with its default settings, on the top 25 candidates from hybrid search (Qwen3 ran on only 100 claims, as described below). The wording of Jev's question was tuned on 100 training claims, and these models had no tuning at all.

| reranker | licence | parameters | nDCG@10 | change vs Firn alone (95% interval) | seconds per search |
|---|---|---|---|---|---|
| Firn alone | | | 0.727 | | |
| MS MARCO MiniLM-L12 | Apache-2.0 | 33M | 0.707 | -0.020 (-0.047 to +0.007) | 1.7 |
| jina-reranker-v1-turbo | Apache-2.0 | 38M | 0.753 | +0.025 (+0.002 to +0.049) | 1.2 |
| bge-reranker-base | MIT | 278M | 0.725 | -0.002 (-0.026 to +0.023) | 5.3 |
| gte-reranker-modernbert-base | Apache-2.0 | 150M | 0.780 | +0.053 (+0.026 to +0.080) | 9.8 |
| Jev, direct | API | | 0.797 | +0.070 (+0.043 to +0.096) | 0.4 (API) |

All 300 test claims. The open source timings are from a laptop CPU with no GPU, and Jev's is the API call time.

The first two models I ran, MiniLM and jina, were less convincing than Jev: MiniLM scored below Firn alone and jina gained 0.025. If I had stopped there, I would have concluded that free rerankers add little. bge-reranker-base, the largest model in the table at 278M parameters, made no difference either, and model size did not explain the pattern: gte-reranker-modernbert-base, at 150M parameters, did far better.

gte got most of the way to Jev: 0.780 against 0.797. The gap of 0.017 could be zero (interval -0.005 to +0.039), but this is one dataset and 300 claims, so I would not call it a match.

The cost moves from money to time. On a CPU, gte took 9.8 seconds per search against 0.4 seconds for the Jev API. A GPU would change that, but I did not measure one.

Qwen3-Reranker-0.6B (Apache-2.0) took 32.5 seconds per search on the same CPU, over three times as long as gte, so I ran it on a random 100 claims only. That sample is easier than the rest, so the numbers do not compare with the table. On those claims Firn alone scored 0.782, Qwen3 0.814 and Jev 0.829.

## Where I ended up

These are my results, from my own setup on one dataset, and they are not a TypeSafe benchmark. SciFact is public and I cannot see what Jev was trained on, so I cannot rule out that it has seen these abstracts, and the same applies to every model I tested. I also measured latency for one user at a time.

Jev improved Firn's ordering. Of the open source models I ran on all 300 claims, gte came closest. Its numbers were encouraging, but ten seconds per search on a CPU is too slow to pursue further as an option I'd build into Firn.

I'm leaving reranking out of Firn itself for now. Someone deploying Firn could still add that step after retrieval, using a hosted service such as Jev or running an open source model on their own hardware.