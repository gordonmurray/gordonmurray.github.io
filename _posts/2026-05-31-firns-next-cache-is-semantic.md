---
layout: post
title:  "Firn's Next Cache Is Semantic"
date:   2026-05-31 09:00 +0100
categories: data
tags: ["rust", "lance", "foyer", "s3", "vector", "search", "cache", "clip", "firn"]
---

[Firn](https://gordonmurray.ie/data/2026/05/02/s3-is-the-perfect-place-to-store-data-until-you-try-to-search-it.html), my object-storage-backed vector search engine, already caches query results. The catch is that it only helps when a query comes back exactly: same namespace, same parameters, same bytes. That is fine for dashboards and saved searches, but most retrieval traffic never repeats exactly. One user searches "holiday photos", the next searches "photos of my holidays", and to an exact cache those are two different requests even though the vectors almost overlap.

I wanted to make those near-miss queries better without changing what they return. This is a post on two attempts at resolving that.

## Lance's CacheBackend

Firn is built on LanceDB and Lance. Lance added a pluggable `CacheBackend` trait for index state on 2026-03-29 ([PR 6222](https://github.com/lance-format/lance/pull/6222)). That is the kind of hook an object-storage-backed search engine wants. Instead of caching final query results, cache the internal building blocks Lance needs for every search: index metadata and decoded index state.

If that worked, a novel query would still run through Lance, so search semantics would stay exact for the chosen index. But repeated object-storage reads for warm namespaces could come from local cache instead of S3.

Firn already uses foyer for a RAM-plus-NVMe result cache, so I tried wiring the two together:

1. Upgrade to a Lance/LanceDB version with `CacheBackend`.
2. Confirm lancedb can accept a custom Lance session from Firn's actual connection path.
3. Re-run the object-storage correctness tests after the dependency upgrade.
4. Implement a foyer-backed `CacheBackend`.
5. Benchmark novel-query latency with and without the new index cache.

The plumbing worked: the custom backend was reachable through Lance sessions, and counters moved during index build and query execution.

## The Cache Worked, but.. didn't perform

The benchmark used a 1M-row namespace with 1536-dimensional vectors, `k=10`, the default `nprobes=20`, and 1000 novel queries. It was an in-process run against the local docker-compose MinIO stack, not real S3. On this local MinIO benchmark, object-store round trips are effectively local and the host page cache is hot. Against real S3, the same sequence may be hundreds of milliseconds if it fans out into many GET or range GET calls.

The point was narrower: measure whether this lower-level index-cache shape helped different queries against the same namespace.

The baseline, current Firn without the foyer-backed Lance index cache, looked like this:

| mode | p50 | p95 | p99 | max |
| --- | ---: | ---: | ---: | ---: |
| current Firn | 9.77 ms | 19.15 ms | 32.09 ms | 47.84 ms |

The foyer-backed `CacheBackend` path was:

| mode | p50 | p95 | p99 | max |
| --- | ---: | ---: | ---: | ---: |
| warm index cache | 26.14 ms | 37.21 ms | 45.76 ms | 54.63 ms |

That is more than 2x slower at p50, well past a borderline miss. The cache made the workload slower.

The cache was doing work. During the measured window, Lance made 29,000 `get` calls, 20,000 `get_or_insert` calls, and around 85 `insert` calls after warmup. Lance was finding most of the index state in the cache.

The bottleneck was data representation. Lance's default memory cache stores ready-to-use objects, so a hit is almost free, mostly a pointer clone. My foyer-backed adapter stored serialized bytes so it could spill to disk, which meant every hit had to copy those bytes and deserialize them back into objects before Lance could use them.

At this scale, the index state fits comfortably in RAM. Adding a disk-spill layer introduced serialization overhead that outweighed any benefit. The issue was never whether the cache was reachable; it was the shape of the value stored in it.

That shelves this specific adapter shape. A serializing cache that pays codec cost on every in-memory hit lost to Lance's native decoded cache on a clean local benchmark, and that finding is portable: the deserialize tax is paid per cache hit regardless of where the object store sits. What MinIO does not answer is whether production cold-query pain is storage-roundtrip-bound in the first place. If real-S3 cold queries are dominated by object-storage latency, a different cache shape may still be worth building: memory-first with disk overflow, or a byte-range cache under `object_store`.

## Semantic Caching

Semantic caching is not a fix for that `CacheBackend` result. It is a different approach.

`CacheBackend` approach: can a novel query stay exact but run cheaper because Lance's index state is already warm?

Semantic caching approach: can a near-duplicate query skip the backend entirely because a very similar previous result is good enough?

Semantic caching is an optional extra on top of exact cache.

The read path is ordered like this:

1. Check the exact result cache.
2. If that hits, return it.
3. If it misses and semantic caching is not enabled, run Lance as before.
4. If it misses and semantic caching is enabled, look for a very similar previous query in the same namespace generation.
5. If the similarity threshold passes, reuse that previous result set.
6. Otherwise run Lance, return the exact result, and remember this query as a future semantic-cache candidate.

Exact search remains the default. The API makes the tradeoff explicit:

```json
{
  "vector": [0.1, 0.2, 0.3],
  "k": 10,
  "nprobes": 20,
  "semantic_cache": {
    "enabled": true,
    "min_similarity": 0.995 <---- this is the value to determine "how close"
  }
}
```

That tells Firn that if this exact query misses, it may reuse a result from a previous query whose vector is extremely close.

Firn is multi-tenant, and that block rides on each individual query, so the control is as fine-grained as you need. Every namespace is an isolated tenant with its own object-storage prefix, and the semantic cache follows the same boundary: one bounded list per namespace, so a reused result never crosses from one tenant into another. The threshold itself is a per-request value. You can enable it for one tenant and leave it off for another, set a relaxed threshold for cheap browse queries and a strict one where the ranking has to be right, or flip it on for a single request and off for the next, with no server-side configuration to change.

Because the threshold is a calibration job and not a fixed answer, Firn exposes three Prometheus counters for the sidecar: hits, misses, and rejections broken down by reason. You can watch your own hit rate against your own embedding model and corpus and move the threshold from there.

## What the Benchmarks Show

I ran this at three levels, each more realistic than the last: synthetic near-duplicate vectors, real CLIP text paraphrases against a small image corpus, and a larger calibration on COCO, a standard public dataset of images with human-written captions. Two things came out of it.

First, when a semantic query hits, it is cheap, meaning it answers from memory in microseconds and skips the object-storage round trip a normal query needs. On the synthetic run a hit landed close to exact-cache latency and about 223x faster than running the query against the backend on this local MinIO setup. The read path was never the question.

Second, and this is the one that matters: real paraphrases often do not clear the safe default. With CLIP text embeddings, natural pairs like "messy desk" and "cluttered desk with papers" usually sit below 0.995 cosine even when they obviously mean the same thing. At the default threshold the cache stays quiet and misses, which is the correct, conservative behaviour.

The COCO calibration shows that tradeoff directly. I used 80 caption pairs, two human captions per image, as a proxy for the same intent expressed in different words. At the conservative thresholds nothing hit. Lowering the bar bought hit rate at the cost of quality:

![COCO semantic cache threshold curve](/images/semantic-cache-coco-threshold-curve.svg)

| threshold | hit rate | mean top-10 overlap on hits | p10 overlap on hits |
| ---: | ---: | ---: | ---: |
| 0.950 | 3.8% | 80.0% | 60.0% |
| 0.900 | 18.8% | 78.7% | 60.0% |
| 0.850 | 36.2% | 80.0% | 60.0% |
| 0.800 | 53.8% | 77.2% | 60.0% |
| 0.750 | 73.8% | 71.2% | 50.0% |

At 0.85 the cache hit about a third of the pairs and kept the target image in the reused top 10 every time. Push the threshold down to 0.75 and the hit rate climbs to 73.8%, but the worst case gets ugly: one hit had 0/10 overlap with the true result. So keep 0.995 as the safe default, and treat lower values, somewhere around 0.90 to 0.85 for this CLIP workload, as an application choice that needs calibration for your own model and corpus.

The holiday-photo example from the opening is one of these cases. "Holiday photos" and "photos of my holidays" are the same request to a person, but their CLIP text vectors can sit well below 0.995, right in the band where the safe default stays quiet and misses. Semantic caching can serve that pair from cache, but only if the application lowers the threshold on purpose and accepts that the reused top-k is approximate. There is no setting that makes paraphrase matching both exact and free.

## Why Opt-In Matters

Semantic caching is approximate by design. Two queries with very similar vectors usually return similar results, but that is not guaranteed, so reusing one result for the other is a judgement call the application has to make, not a decision Firn should make for you. That is why it stays off unless you ask for it. The same caution is why this first version only handles single-vector search; multivector, text, and hybrid queries cannot use it yet.

If you want exact results, do nothing. Firn behaves as it always has.

If you have a workload with repeated intent, and you care more about latency and storage cost than recomputing the exact results every time, you turn semantic caching on for those requests.

The `CacheBackend` experiment that did not pay off cleared the deck for this. It showed that a cache can be wired up correctly and still be the wrong tool for the job. Semantic caching is a different approach for a different problem: rather than trying to make every novel query cheaper down in the storage layer, it gives the application a simple switch for the cases where "close enough" really is close enough.
