---
layout: post
title: "Cloudfloe's query engine in a celld cell"
date: 2026-09-05 09:00 +0100
categories: data
tags: ["cloudfloe", "celld", "duckdb", "iceberg", "wasm", "s3"]
---

[Cloudfloe](https://github.com/gordonmurray/cloudfloe) is a web interface I made a while back for querying Apache Iceberg data on S3 with DuckDB. A user saves a connection, writes SQL, runs it, and can see the results. In the hosted version, a query session starts a container containing native DuckDB to run the queries. It’s not overly innovative but it works well.

As an experiment I wanted to try running DuckDB inside [celld](https://github.com/denoland/celld) and see if there were any interesting results, knowing that it’s probably not a typical use case for celld.

Celld is a self-hosted implementation of Cloudflare Workers and Durable Objects. A named Durable Object is a cell with its own SQLite database. Application code runs in a V8 isolate. Long-lived state is replicated through an object store, which also acts as the coordination layer for ownership and failover.

At first glance, celld sounds like a faster place to start a query worker. That was the hope anyway. Celld does not run containers. It can’t start the native DuckDB binary that Cloudfloe uses. Trying celld means compiling DuckDB to WebAssembly and running it inside a single-threaded V8 isolate. That’s how it started. A number of interesting things cropped up from there.

## DuckDB was already halfway there

I started with [Ducklings](https://github.com/tobilg/ducklings/tree/v1.5.5), a compact DuckDB WebAssembly build for browsers and serverless runtimes. It runs without threads and uses Asyncify to let DuckDB pause for an asynchronous `fetch()` and resume when the data arrives.

The part I expected to get stuck on was Iceberg. Reading Parquet files is only part of reading an Iceberg table. Something also has to interpret the table metadata and Avro manifests to work out which files belong to the snapshot being queried.

I thought I would have to compile DuckDB's Iceberg extension into the module myself, or do that metadata work in JavaScript and hand DuckDB a list of Parquet files. But the Ducklings Workers package I checked already included `httpfs`, Avro, and Iceberg. Someone had done the first option already.

That gave me a much shorter route to the first test: could celld load this build and run a query?

Before getting to Iceberg, I needed to know whether the runtime would accept a database engine this size. The DuckDB WASM file alone was 42.4 MB uncompressed. These are deployment sizes, not the memory DuckDB needs to run a query. A bundle-size limit could stop the experiment before it started.

Celld loaded the module, and DuckDB returned its version and a SQL result. That answered the first question: the existing build could run. Next up was a query against an actual Iceberg table.

## A real Iceberg query worked

I created a 37,537-row Iceberg v2 table with PyIceberg and stored it in MinIO. There was one small setup issue: DuckDB expected a `version-hint.text` file when I gave it the table directory, but PyIceberg had not created one. Giving DuckDB the exact location of the metadata JSON file fixed that.

From there it worked. DuckDB inside the celld cell read the table metadata, decoded the Avro manifests, read the Parquet data, and returned the correct row count. The count query took 148 ms, with celld and MinIO running on the same machine. Cloudfloe's basic query path was now running inside a celld cell.

An equality-delete test returned the expected remaining rows too. That gave me a little more confidence than a count alone, though there were still Iceberg features I had not tested.

The next question was how much time I was spending getting DuckDB ready to do any work. The 148 ms result alone would not tell me that. Later tests would show a trivial `SELECT 42` taking over 300 ms in a fresh celld process, compared with about 3 ms when the cell kept DuckDB open. Getting the engine ready would turn out to be a large part of the wait.

## What happens when the cell goes to sleep?

For Cloudfloe, there would be gaps between a user's queries. I wanted to know whether a cell could go to sleep between requests and still behave correctly when it woke up.

First I tested what happened if celld was asked to evict a cell while DuckDB was waiting for data. I delayed a Parquet response for two seconds, then requested eviction while the query was still running. The query finished correctly in about 2.1 seconds. Celld let it finish before putting the cell to sleep.

After the cell had hibernated, I sent another request. Celld reconstructed it, restored a request counter I had saved in its SQLite database, and ran the DuckDB query again. The saved state survived, and the query worked after waking.

That part worked. The next thing to check was how much memory going to sleep actually gave back.

## Putting the cell to sleep didn’t give all the memory back

I used `ps` to measure the celld process's resident set size (RSS), the amount of its memory currently held in physical RAM. This measures the whole process, including the runtime and DuckDB. I recorded it before a query, after the query, and after hibernation, then repeated the test in a fresh celld process. I also checked celld's internal state to confirm that the cell and its isolates had been released.

The rounded numbers I noticed were:

- about 140 MB for the clean celld process in this setup;
- about 230 MB after its first DuckDB query;
- about 95 MB added by that first query;
- about 210 MB after the cell and isolates were freed.

The cell was gone, but celld was still using about 75 MB more RAM than it had before the first query. That does not necessarily mean a memory leak. A process can hold on to memory for reuse rather than return it to the operating system immediately. What I could see in both runs was that putting the cell to sleep did not bring memory usage back to where it started.

Inspecting the Ducklings build also showed a separate constraint: DuckDB's WebAssembly memory was capped at about 134 MB, with no disk fallback for queries that needed more working space. Raising that limit would require rebuilding Ducklings. That made the amount a query held in memory more important than the table's total size, but I had not tested where different queries would hit the limit.

Putting the cell to sleep had released some memory, but the process was still well above where it started. On the next request, DuckDB would need to be opened again anyway. I wanted to measure how much that restart cost.

## What does waking up a cell cost?

I used a query that just returned a number and read no tables. That meant the request could spend no time fetching Iceberg data, making it easier to see the cost of getting DuckDB ready.

The full request took roughly 320 ms in a fresh celld process. After putting the cell to sleep and waking it again, with celld itself still running, it took about 250 ms. Keeping the cell's DuckDB database and connection open brought repeated requests down to about 3 ms. These were all local tests on the same machine.

I timed the startup stages separately to see where that difference came from. Waking a cell meant creating its runtime and opening DuckDB again. The SQL itself was only a small part of the work.

Keeping DuckDB open made repeated queries much faster. Putting the cell to sleep meant doing much of that setup again, even though the process still held on to memory. Could I initialize DuckDB once and use that as a starting point for cells waking up?

## Could I make a cell template?

I was thinking along the lines of a custom Docker image. You install the dependencies once and use that image as the starting point each time. Could I do something similar for a cell, with DuckDB already initialized?

That would need to go a step further than preinstalled software. I wanted to preserve the work done when DuckDB starts, so a new cell would not have to repeat it. I looked through celld's documentation and runtime source to see whether it could save and restore something like that.

Celld already avoids repeating one expensive step. A statically imported WASM module is [compiled once per process and reused by later isolates](https://github.com/denoland/celld/blob/v0.4.0/docs/wasm.md). So the startup time I was measuring did not include compiling the 42.4 MB DuckDB module every time.

But I found no template containing an initialized DuckDB. Celld's storage snapshots preserve SQLite state, not the JavaScript objects or WebAssembly memory of a running query engine. Restoring the cell's saved data still left the runtime and DuckDB to be initialized.

There was another useful clue. A cell and an isolate are different things: the cell is the named object, and the isolate is the V8 runtime hosting it. Several cells from the same Worker can share an isolate. In one test, a new cell placed in an already warm isolate activated in about 1 ms. It still needed its own DuckDB database, but Ducklings was already initialized. Part of the starting point I wanted was already there.

That suggested a smaller experiment. When the last cell in an isolate went to sleep, could celld keep the empty isolate around for the next request? It would preserve the initialized runtime, though each returning cell would still have to open its own DuckDB database.

I made a local celld patch that kept at most one empty isolate per Worker. It reused an existing isolate rather than creating a snapshot or making copies of a template. I deliberately used a one-second idle timeout to make cells hibernate during the test; celld's timed idle eviction is disabled by default.

The next request took about 145 ms with the isolate retained, compared with 254 ms when celld had to create a new one. Reusing the runtime skipped Ducklings initialization, though the reconstructed cell still had to open its own DuckDB database.

Keeping the isolate alive worked. It saved roughly 100 ms on the next request, but it also kept more memory in use. After the first hibernation, the process used about 243 MB with the isolate retained, compared with 218 MB when it was released. Repeated sleep-and-wake cycles pushed memory higher, though the short test was not enough to call that a leak.

So I had a useful tradeoff: faster wakes in exchange for more resident memory. Keeping isolates around indefinitely would need a memory budget, especially as the number of Workers grew. The next thing to try was keeping one alive briefly, then releasing it if no more queries arrived.

## What if I kept a cell warm for ten seconds?

I changed the patch so an empty isolate could stay alive for ten seconds after its last cell went to sleep. That seemed like a useful window for someone editing SQL and running another query. If they stopped, celld could release the isolate.

This time I used the actual Iceberg count query. Each request opened a new DuckDB database, ran the query, and closed the database and connection afterward. A request arriving within the warm window completed in about 93 ms. After the isolate had been released, it took about 291 ms. The correct row count came back in both cases.

So the short window worked too. In this local test it saved about 200 ms on the full request. But an expiry time only answers how long to keep something when there is room for it. I also needed to know what happened when the node needed that memory sooner.

[Watch the terminal recording](https://asciinema.org/a/BkWvvkyPAhfN5HxG): the same Iceberg query from a fresh process, with the runtime retained, and after it expires. Each run returns the same 37,537 rows. This is a separate recorded run, so its timings differ from the measurements above.

## What happens when the node needs the memory back?

For a separate test, I deliberately lowered celld's memory-pressure threshold to about 210 MB and extended the warm window to thirty seconds. This forced the memory policy to act while an empty isolate was still being retained. The first query succeeded, then celld put the idle cell to sleep and stopped admitting new work because memory usage was too high. My patch kept the empty isolate alive anyway. The next query waited until the client's five-second timeout expired.

The isolate was eventually released, about 35 seconds after it became empty. Even then, the process held on to enough memory that celld stayed in its memory-pressure state. Twenty seconds later, there were no resident cells left to evict, but the node was still not admitting work.

The low threshold was deliberate. This did not show that a normal celld deployment would get stuck after one query. It showed that my retention patch and the node's memory policy did not work together: the patch waited for its timer, and releasing the isolate still did not reduce memory enough for the node to resume.

Keeping the runtime warm had made queries faster. Making that useful needed more than a timer. Memory pressure had to be able to end the warm window early, and the node needed a way to recover or return a clear error when there was nothing left to evict.

I started this wondering whether Cloudfloe could run its queries in a celld cell. It could. What I hadn’t expected was to spend so much of the experiment looking at what happened between queries.

Keeping part of the runtime alive made the next query faster. Putting a timer on it helped, but the memory-pressure test showed why that wasn’t the whole solution. There seems to be something worth exploring here: keeping expensive runtimes ready while still letting the node recover memory when it needs to.

I’m leaving Cloudfloe’s containers in place for now. This was an experiment with a workload celld probably wasn’t designed around, and I had fun seeing how far it would go. Getting DuckDB to run was the beginning. Working out what to keep alive afterward turned out to be the interesting part.

## Test setup and reproduction notes

These tests ran in September 2026. The timings describe this local setup, not production performance.

- **Software:** [celld 0.4.0](https://github.com/denoland/celld/tree/v0.4.0), [Ducklings 1.5.5](https://github.com/tobilg/ducklings/tree/v1.5.5), and DuckDB 1.5.5. Initial tests used the official celld binary. Isolate-retention experiments used my local patches, not a built-in celld feature.
- **Machine and data:** Linux x86-64, Intel Core i7-13700H, 20 logical CPUs, about 15 GB RAM. Celld, MinIO, and the client ran on the same machine. The main fixture was a 37,537-row Iceberg v2 table created with PyIceberg; queries used its exact metadata JSON location.
- **Deployment:** Directly with `celld deploy`. The optional managed deployment route had a module cap of about 26 MB, below the 42.4 MB DuckDB module.
- **Lifecycle settings:** `CELLD_IDLE_EVICT_S=1` forced frequent hibernation; timed idle eviction is disabled by default. My first patch used `CELLD_RETAIN_EMPTY_CELL_ISOLATES=1`. The timed version used a reserve of zero and `CELLD_EMPTY_CELL_ISOLATE_GRACE_S=10`. These retention settings belong to the experimental patches. Maintenance checked expiry periodically, so ten seconds was not an exact release deadline.
- **Pressure test:** A thirty-second grace and `CELLD_MAX_RSS_MB=200` deliberately triggered pressure at about 210 MB. The policy required memory to fall below 80% of that threshold, about 168 MB, before accepting new work again. It did not reach that point during the observation period. This was a forced test, not the default configuration.
- **Measurement:** Client timings included routing, cell activation, initialization, and query execution. Separate stage timers and a random module ID distinguished fresh and reused runtimes. RSS measured the whole celld process. MB figures are decimal and rounded independently; the short runs do not establish long-term memory growth.
- **Timed Iceberg queries:** Every request opened and explicitly closed its DuckDB database and connection. Control and treatment used the same custom Ducklings build, with Arrow insertion support removed. That reduced raw bundle size by only 0.21%.
- **Coverage:** The v2 count and a separate equality-delete fixture passed. Positional deletes, v3 deletion vectors, time travel, and complex field-ID schema evolution were not verified. These tests do not establish a safe maximum table size.
