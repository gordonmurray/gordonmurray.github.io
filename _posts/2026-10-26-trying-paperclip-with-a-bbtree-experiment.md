---
layout: post
title: "Trying Paperclip with a bbtree experiment"
date: 2026-10-26 09:00 +0000
categories: data
tags: ["paperclip", "bbtree", "firn", "rust", "codex", "benchmarks"]
---

I came across [bbtree](https://github.com/XiangpengHao/bbtree), a Rust ordered-map library with some substantial performance claims. Faster operations, lower memory usage, and a concurrent implementation. I wondered whether it might be useful somewhere in [Firn](https://github.com/gordonmurray/firnflow), my vector and full-text search engine.

An ordered map stores key/value pairs and lets you look up a key or scan a range in order. It is a fairly small piece of a storage system, but one that can get called a lot.

I also wanted to try [Paperclip](https://github.com/paperclipai/paperclip), which coordinates coding agents through projects and tasks. Evaluating bbtree seemed like a reasonable first job for it. Research the library, build some benchmarks, check correctness, and write up what happened. That was the plan anyway.

## Getting a project running

I ran Paperclip locally and used its web UI to follow the work. The bbtree project had a PM agent and a planning task pointing at a `PLAN.md` with the scope and constraints. From there, specialist agents took on implementation, testing, and review.

I liked being able to see the tasks and who was responsible for them. It gave me somewhere to check progress without having to piece everything together from terminal conversations.

I wanted to check it from my phone too. Codex changed Paperclip from localhost-only access to its authenticated LAN mode. That worked, although the restart interrupted an active PM run. Work continued afterward, but that particular interruption was down to changing the setup while it was busy.

## Keeping it running was harder

The project stopped a few times. The first task was blocked because the workspace had no Git metadata. That was straightforward to fix.

The more frustrating failure came later. One agent changed and committed files in the shared checkout while another was running benchmarks. The benchmark checks noticed that the source revision had changed and refused to continue.

I wanted those checks. Comparing measurements from different revisions would have made the results difficult to trust. But I had expected the PM to arrange the work so that agents weren't changing the code underneath a running test.

Another attempt stopped because background CPU activity exceeded the benchmark's threshold. This was my laptop, and the runner sampled CPU load for only 250 milliseconds. It could reject a run without waiting for the machine to settle and trying again.

From the UI, it looked like the project had stopped making progress. Paperclip's server was still running; the benchmark jobs were blocked. I found myself checking in and asking the PM to continue more often than I expected.

Eventually I asked Codex to check every 30 minutes and help if something was stuck. It used a local scheduled monitor with authenticated access to Paperclip. It could inspect tasks and send PM a recovery message, but it stayed out of the benchmark checkout.

At 22:32 it sent a targeted nudge and woke PM. At the next check, the remaining review work was complete. I had ended up with Codex watching Paperclip coordinate the agents. More supervision than I had planned, but it helped get the project over the line.

## The first report needed another pass

An independent reviewer found problems with the evaluation itself. One concurrent-scan test could miss incorrect output. The analysis called some noisy comparisons conclusive, despite the project's own rules. The memory evidence was also missing from a clean checkout, so someone else couldn't reproduce the report from the repository alone.

The agents fixed the tests, corrected the analysis, and packaged the evidence. A second review approved the result. Without that review step, I would have had a finished-looking report with claims it couldn't fully support.

## Faster depended on what I asked it to do

The tests compared bbtree's single-threaded map with Rust's standard `BTreeMap`, and its shared map with `SkipMap` and a `BTreeMap` protected by a read/write lock.

In the single-threaded tests with integer keys and values, inserting new entries achieved about 4.7 times the throughput of standard `BTreeMap`. Removing existing entries achieved only about 30% of its throughput. Successful lookups favoured bbtree; missing-key lookups favoured the standard map.

Shared bbtree also did well in the selected eight-thread mixed workloads, with about 2.7 to 4.3 times the throughput of `SkipMap`. Some other comparisons stayed too noisy to draw a conclusion, and the corrected report leaves them inconclusive.

So I couldn't take the insertion result and apply it to everything a map does. The mix of operations mattered.

## Empty didn't mean the memory was released

The memory result was the one I kept coming back to.

With 100,000 integer keys and values, shared bbtree used about 31% less allocated memory than the locked standard map while full. After deleting every entry and waiting one second, it still retained roughly 2.36 MB of allocation. The map was empty, but it was still alive.

Dropping the entire map released that memory. This measured live allocations, rather than how much physical RAM the operating system assigned to the process. It wasn't evidence of a permanent memory leak.

A buffer that fills up, gets processed, and is discarded has a different lifetime from a map that stays alive for the duration of a service. I would want to test repeated insert/delete cycles before putting the shared map into a long-running, memory-constrained workload.

## Where that leaves Firn

This project evaluated bbtree on its own. It never put bbtree into Firn or measured search or ingestion performance. Firn was the reason for looking, but the experiment stayed at the library level.

The results are from one Intel i7-13700H Linux machine, using [bbtree at this revision](https://github.com/XiangpengHao/bbtree/tree/8adae97a0ed97e5c98fa2790312eb0aec9719846). Performance tests used integer keys and values. Memory and correctness tests covered additional shapes. None of that establishes a speedup for Firn.

I'm leaving Firn unchanged. Before trying an integration, I'd profile a real workload and find out whether an ordered map is taking enough time to be worth replacing. Then I'd test that path with Firn's actual data.

For another Paperclip project, I'd arrange isolated checkouts and a quiet benchmark window from the start, and give the PM an explicit recovery policy for blocked work. I'd keep the independent review too. It caught things I would have missed by watching tasks turn green.
