# Building a Digital Asset Management (DAM) project with different tools

## Introduction

I wanted to build a pipeline for digital asset management (DAM) that could handle image uploads, process them asynchronously, and enrich them with some ML modesl.

I also wanted to avoid using a typical database for this sort of project. I want to use some new tools or technologies that I either haven't used before (like fly.io and Cloudflare) or dont get a chance to use as much as I'd like (such as using Parquet for data).

 This post is both a technical write-up and a personal diary entry of sorts of what I created so far, why I built it, and where I want to go next.

## Architecture Overview

The project is split into two main components:

- **Producer (FastAPI):** Handles image uploads, stores them in Cloudflare R2 (S3-compatible object storage), and queues metadata to RabbitMQ provided by CloudAMQP. I used R2 instead of S3 since it is currently cheaper and an excuse to use some of Cloudflares services.
- **Worker (Python):** Listens to the Rabbit queue, downloads images, generates captions with BLIP, vectorizes images with CLIP, and appends metadata to a Parquet file in R2.
- **Learning SvelteKit:** I decided to use SvelteKit to build a minimal front end, even though front end development isn't a strength of mine. This was an opportunity to step out of my comfort zone and learn a new framework.

The componets all run on Fly.io and stay mostly within the free tier. The Worker keeps failing due to low memeory but easily solved by scaling up a little. CloudAMQP is new to me too. It was quick and easy to get a rabbitmq running and provided a nice UI to keep an eye on the queues.

The data flow looks like this:

<div class="mermaid">
flowchart LR
    Client[Client]
    FastAPI[FastAPI Producer]
    RabbitMQ[RabbitMQ]
    Worker[Worker]
    R2[R2: assets + metadata + vectors]

    Client --> FastAPI
    FastAPI --> RabbitMQ
    RabbitMQ --> Worker
    Worker --> R2
</div>

The resulting files and flders on R2 look like:

```
r2-bucket-root/
├── assets/
│   ├── b227facd-7e87-4988-9a64-6f728123f79c.jpg
│   ├── 8a1c2d3e-4f5b-6789-0abc-def123456789.png
│   └── 9f8e7d6c-5b4a-3210-fedc-ba9876543210.jpeg
├── vectors/
│   ├── b227facd-7e87-4988-9a64-6f728123f79c.json
│   ├── 8a1c2d3e-4f5b-6789-0abc-def123456789.json
│   └── 9f8e7d6c-5b4a-3210-fedc-ba9876543210.json
└── metadata/
    └── images.parquet
```

* assets/ contains the original uploaded image files, named by UUID.
* vectors/ contains the CLIP vector representations for each image, stored as JSON files with matching UUIDs.
* metadata/images.parquet is a Parquet file that accumulates all metadata (filename, caption, vector key, etc.) for easy querying and analysis.

The resulting data in Parquet format sotred on R2 looks like this:

```
{
  "id": "b227facd-7e87-4988-9a64-6f728123f79c",
  "filename": "image.jpg",
  "r2_key": "assets/b227facd-7e87-4988-9a64-6f728123f79c.jpg",
  "vector_key": "vectors/b227facd-7e87-4988-9a64-6f728123f79c.json",
  "mime_type": "image/jpeg",
  "caption": "a golf ball on the tee at the golf course"
}
```

## Tools/technologies

- **FastAPI** for the API layer
- **CloudAMQP RabbitMQ** for async message queuing
- **Cloudflare R2** for object storage
- **PyTorch CLIP** for image vectorization
- **BLIP** for image captioning
- **Parquet** (via pandas/pyarrow) for efficient metadata storage

## What I Built

- **Image Upload API:** A user can upload images via a simple HTTP endpoint.
- **Async Processing:** Images are stored and queued for background processing, decoupling upload from analysis.
- **ML-Powered Metadata:** Each image is captioned (BLIP) and vectorized (CLIP), with results stored alongside the original file.
- **Structured Metadata:** All metadata is appended to a Parquet file, making it easy to analyze or query later.

## Lessons Learned

- **R2:** Cloudflare R2 was easy to use and low cost. Im used to using S3.
- **Some ML:** Integrating CLIP and BLIP was surprisingly smooth with Python.
- **Parquet:** It's a great effecient format for storing and querying structured data.
- **Fly.io** - Really easy to use. I keep typing "docker" instead of "fly" in the command line to manage the app or view the logs.
- **cloudamqp.com** - Also easy to use. I had a rabbitmq queue ready in a minute or two. And a new UI via LavinMQ which is new to me.

## Future Directions

This project is a mini milestone, but it's also a starting point. Here's what I want to explore next:

- **Searching Parquet Files:** Investigate the feasibility of searching within Parquet files directly. While Parquet is efficient for storage and querying, it may not be optimized for complex search operations. Pros include reduced storage costs and fast read times for large datasets. Cons might involve the need for additional indexing or external search systems to handle more sophisticated queries efficiently.
- **Model Serving with BentoML:** Instead of running CLIP and BLIP directly in the worker, I want to try serving models with BentoML for better scalability and modularity.
- **Svelte UI Gallery:** Build a Svelte-based frontend to display a gallery of uploaded images, complete with thumbnails and metadata.
- **File History & Edits:** Add features to show the history of edits or metadata changes for each file, by reading and visualizing the Parquet data.
- **More Asset Types:** Extend the pipeline to handle other asset types (audio, video, documents) and enrich them with relevant ML models.

## Conclusion

I'm happy with what I've built so far. Its minimal but useful for future experiments in digital asset management and ML integration.

---

* This post is as much for future-me as it is for anyone else!
* Code on [GitHub](https://github.com/gordonmurray/dam-pipeline-fastapi-clip)