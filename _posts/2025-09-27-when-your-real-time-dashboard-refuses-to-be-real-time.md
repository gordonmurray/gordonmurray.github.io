---
layout: post
title:  "When Your Real-Time Dashboard Refuses to be Real-Time: An Evening with Rill, Paimon, and DuckDB"
date:   2025-09-27 21:00
categories: data
tags: ["apache","flink","paimon", "cdc", "duckdb", "rill"]
---

I sat down with the kind of confidence that only a a healthy CDC pipeline can deliver. MySQL was passing on changes to Flink, Flink was talking to Paimon, and Paimon was sending Parquet files neatly into MinIO. All I wanted was a nice Rill dashboard I saw online to add some visualisations to the data.

## Hour 1: Optimism
Connecting Rill to DuckDB and pointing it at those Parquet files worked straight away, 26 sample products appeared instantly. Then I added more data to MySQL. Thirty-three records now. Paimon? Also thirty-three thanks to Flink. Rill clearly preferred the memory of a simpler time. A couple of refreshes, kicking containers and even adding more records and waiting. The dashboard still clung to 26 records total.

I convinced myself it was a cache hiccup. DuckDB has an in-memory mode, so I tried `path: ":memory:"` but no joy.

## Hour 2: Its always Cache
If the cache wasn't playing nice, maybe Rill needed a more direct relationship with MinIO. I swapped the model to raw SQL with `read_parquet('s3://warehouse/cdc_db.db/products_sink/**/*.parquet')`. MinIO answered with a stack of `HTTP 400 Bad Request` errors and a lecture on AWS regions. I fed DuckDB every S3 setting I could think of: endpoint, SSL flags, URL styles, keys... both in env vars and `SET` statements. DuckDB's CLI nodded approvingly; Rill didn't budge.

![Rill screenshot]({{ site.url }}/images/rill_screenshot.png)

## Hour 3: YAML Lies
The logs were full of “cannot be reconciled…” errors. Reading Documentation finally paid off. The SQL isn't supposed to live inside `.yaml` resource files; it wants its own `.sql` file in `/models/`. I moved the query into `paimon_products.sql`, Rill reconciled, and immediately complained that `paimon_data` didn't exist.

DuckDB graciously suggested a fix: try `main8514e79c.paimon_data`. I don't know where that prefix came from but it worked! Until I restarted the container and DuckDB rolled new one: `main6be7e256.paimon_data` this time. Every restart was a random catalog prefix and I couldn't find any way to set a static value or omit it entirely.

## Hour 400: Automation
Instead of resigning myself to manual edits, I scripted a sidecar that called Rill's API, discovered the latest alias, patched the SQL, and touched the model file so reconciliation would notice. There were a few hiccups but the dashboard finally showed live-ish data again. I had automation!

Of course, the Rill Dashboard still stared back with zeros. Rill could see the data in the model but the dashboard was static, they must not be sharing the same source. So the sidecar morphed into a small cron engine, recreating tables, touching metrics files, and sprinkling timestamps every minute just to convince DuckDB new Parquet files existed.

## Stepping Back
I stopped for dinner and the obvious truth landed: DuckDB was never designed to tail a stream of constantly changing files. It isn't Rill's fault. I’d been bending batch tools into a streaming shape because I liked their UI. The pipeline itself—MySQL → Flink CDC → Paimon → MinIO—was solid. The dashboard stack was the mismatch.

## Mental Post-it Notes for Future Me
- DuckDB and Rill are still cool for batch analytics or datasets that change in bursts you can schedule.
- Random catalog prefixes are how DuckDB keeps its attachments tidy; fighting them for real time is an uphill battle.
- If you're writing polling loops and touching files every minute, you may be on the wrong track.

## The Goodbye Lap
By the end of the evening, I had a dashboard that sort-of refreshed, a sidecar that was holding it all together. When you find yourself writing elaborate polling scripts, cache-busting mechanisms, and sidecar containers just to make something "work," step back and ask: *"Am I using the right tools for this job?"*
