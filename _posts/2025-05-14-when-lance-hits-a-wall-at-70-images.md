---
layout: post
title:  "When Lance Hits the Wall at 70 Images on Cloudflare R2"
date:   2025-05-14 13:00:01
categories: data
tags: ["python","vector", "metrics","clip","lance","r2"]
---

First impressions of using [Lance](https://github.com/lancedb/lance) with Python were excellent. It took almost no code at all to upload images, vectorize them, save the data in Lance format in R2 and perform some searches. The resulting files and folders look organised and scalable.

I picked R2 here for lower cost and for a change from working with AWS S3.

The folder structure on R2 after uploading a single file looks something like this:

![Lance R2 folder structure](/images/lance_r2_folder_structure.png)

```
r2-bucket-name/
├── images/
│   └── {sha256}.{ext}               # Original uploaded image (e.g. jpg, png)
├── images.lance/
│   ├── _transactions/
│   │   └── *.txn                    # Transaction logs (append-only)
│   ├── _versions/
│   │   └── *.manifest               # Manifest files for each version
│   └── data/
│       └── *.lance                  # Binary fragments containing vector rows
```

While a few initial images worked I ran in to problems when I wrote a script to add images in bulk. My plan was to add 100, 500, 1000+ images and see how Search performance and accuracy behaved as more files were added.

As I searched I found I was getting duplicates in the search results. Which was confusing as there are no duplicate images since the naming of the files would result in the original file being overwritten. However Lance is an append only database it kept a record of each transaction, which were showing up in the results. I updated the python code to avoid duplicates. That history could be useful later.

```python
existing = tbl.to_arrow().filter(pa.compute.equal(tbl.to_arrow()['id'], img_key))
        if existing:
            logger.info("Vector already exists, skipping insert.")
        else:
            logger.info("Adding vector to table...")
            tbl.add([{
                "id":   img_key,
                "path": f"s3://{R2_BUCKET}/{img_key}",
                "vector": vec.tolist(),
            }])
```

I didn’t get too far adding more data. Turns out I couldn't add any more than 70 images before things came to a complete stop.

At first I thought that the service I was using to generate images (picsum.photos) was rate limiting me, but I ruled that out with logs and found it was the step of adding data to Lance that was unresponsive.

Lance keeps a record of each distinct upload. Every time you call .add() on the table, Lance writes a new transaction. These are stored in /vectors/images.lance/_transactions/ as .txn files.

As far as Im aware this makes the table time-travel possible, each .txn represents a consistent snapshot (like a git commit).

This data takes time to read and update, based on what I could see from the R2 metrics at least, which was very cool. A "lot* of Class A operations in R2.

![Cloudflare R2 Metics](/images/cloudflare_r2_metrics.png)

I have more to learn about the index types that are available but from what I have learned my options are:

* IVF_PQ could be ideal — when you have 10K+ vectors. I couldn't add IVF_PQ index due to not enough records and I couldn’t add more records.

* IVF_FLAT is my only choice right now for low usage, <1K rows

I can't create an index at the very beginning either as there isn't enough data:

```json
{"detail":"Failed to store vector: lance error: LanceError(Index): KMeans: can not train 16 centroids with 0 vectors, choose a smaller K (< 0) instead, /root/.cargo/registry/src/index.crates.io-6f17d22bba15001f/lance-index-0.26.0/src/vector/kmeans.rs:51:21"}
```

With no index in place, I can only add around 60-70 images to R2 before it grounds to a halt.

I can’t add an index at the end either as it runs in to the same slowdown issue as adding new data.

Even though I couldn't add any new data, Search continued to work:

```bash
➜  time curl https://fastapi-lance-r2-mvp.fly.dev/search\?text\=laptop

{"query":"laptop","results":[
{"id":"images/fb4ef5655a362113f11775252c57518f350aa67b675713bfc21891623d4c9a1c.jpg","path":"s3://fastapi-lance-r2-mvp/images/fb4ef5655a362113f11775252c57518f350aa67b675713bfc21891623d4c9a1c.jpg","_distance":1.462988257408142},
{"id":"images/41fec0057b287d2e0fe3f978591da2bbd4e775f292e491bf8a92add53ce4fc11.jpg","path":"s3://fastapi-lance-r2-mvp/images/41fec0057b287d2e0fe3f978591da2bbd4e775f292e491bf8a92add53ce4fc11.jpg","_distance":1.5590983629226685},
{"id":"images/f20866e9bcd9b4b174790314bf6c8a6f75bd5748fcd73171dc65ff5f2f68b4ee.jpg","path":"s3://fastapi-lance-r2-mvp/images/f20866e9bcd9b4b174790314bf6c8a6f75bd5748fcd73171dc65ff5f2f68b4ee.jpg","_distance":1.5672272443771362}]}

 0.03s user 0.00s system 3% cpu 1.075 total
```

I Started over, added 50 images and purposefully stopped. Then I added a IVF_FLAT index using:

```python
    tbl.create_index(
        metric="cosine",
        index_type="IVF_FLAT",
        num_partitions=10, #ideally a square root of the total number of vectors
        vector_column_name="vector"
    )
```

I had all sorts of issues adding an index but it boiled down my column name. my original schema had a field called “vec” and adding an index didn’t seem to like that regardless of want I tried. I cleared out the r2 bucket and started over with a schema with a field called “vector” and it worked.

Here is the output of a /stats endpoint showing the schema of the vector data:

```json
{
  "rowCount": 62,
  "columnCount": 3,
  "columns": {
    "id": "string",
    "path": "string",
    "vector": "fixed_size_list<item: float>[512]"
  },
  "lastVectorDim": 512,
  "tablePath": "s3://fastapi-lance-r2-mvp/vectors/images.lance",
  "indices": []
}
```

The next step was to see if I could successfuly add an additional 50 images.

It failed again at the same spot with the index in place. The last successful image took nearly 20 seconds so the index seems to have helped at least one image, though it failed on the next insert.

```bash
Generating image 13 of 50
Download took: 0.74s
Uploaded 55beb8e8a93551b1e2c27b00ebad0101b610c193ff6ccd8961af464666706984.jpg – Status: 200
Total time: 2.07s
Generating image 14 of 50
Download took: 0.39s
Uploaded 37b09edd9ea531f5a78a860f135cbe2dff2f1a4fa2a0dbf1f23ab9a16c56e32f.jpg – Status: 200
Total time: 19.89s
```

Search still worked at this point.

Looks like R2 isn't the best medium for Lance data due to latency. AWS s3 express zone one might be better but I suspect I’ll hit similar issues there.

Next I plan to try this out with local storage. Hopefully this will allow Lance to query and update the data locally then I will sync the data to R2 periodically allowing searches to use the Lance data on R2 directly and see how that performs.

I’ll create a separate project for that.

If you found this post helpful, Star [https://github.com/gordonmurray/fastapi-lance-r2-mvp](https://github.com/gordonmurray/fastapi-lance-r2-mvp) or sponsor if you're feeling generous!

