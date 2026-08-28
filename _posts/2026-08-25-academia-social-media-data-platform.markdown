---
layout: post
title:  "How we're building academia's largest open-source social media data platform"
date:   2026-08-25 08:00:00 -0700
categories: engineering
pinned: true
mermaid: true
repo: https://github.com/METResearchGroup/lab_data_integrations_interface
---

Data collection can demand more time from an academic researcher than conducting the research itself. APIs exist, but academics are researchers, not software engineers. Academic researchers can spend weeks gathering data just to support a single point in an appendix, only to repeat the entire process for their next project. 

In industry and large research institutions, specialized software engineering teams build complex data pipelines so analysts can access information instantly. Our goal is to bring that into academia.

What we're building is a multi-TB dataset of Bluesky and Reddit posts that researchers can use in minutes, not weeks, eliminating tedious collection work and returning hundreds of hours back to researchers.

* TOC placeholder — kramdown replaces this list with the real table of contents
{:toc}

## Where This Started

When working with Professor William Brady's psychology lab at the Kellogg School of Management, 
a common request from grad students was social media data. Every time, the process required back and 
forth between the engineers (us) and the grad students to communicate and help them get their data. 
By building out this platform, we're providing terabytes of data to users, accessible with just one query.

## What We Tried (and what failed)

At first, we tried using Bluesky's AppView API to ingest data, then ran it through a preprocessing,
feature generation, and curation pipeline. Two things kept it from being a platform:

**1. It wasn't complete.** The AppView API can hand you posts matching certain
keywords or phrases, or the last X events for a given user, but never the last X events from
*everyone*. Every query has to start from something you already know to ask about, which is
backwards for a researcher whose question hasn't been asked yet.

**2. It didn't scale.** Even for the slice we could address, the API caps how fast we're allowed to
pull it. Collecting tens of thousands of records a day already meant pushing into rate limits, and
what we actually wanted was hundreds of thousands, then millions.

## Current Approach

We've split our project into 3 major components:

**1. Gathering live data**

**2. Gathering backfills**

**3. An agentic query engine.**

In this blog, I'll be going over the first, and other blogs will dive into the latter two. 

## Gathering Live Data

Our app forms a websocket connection to Bluesky's Jetstream, where we store rows in a buffer before flushing to object storage.
A cursor keeps track of where we are, so reconnects know where we left off. 

### Architecture Diagram

#### Ingestion

One process reads the firehose, buffers what it parses, and commits batches to Iceberg.

![ingestion system design]({{ "/assets/images/ingestion-architecture.png" | relative_url }})

#### Maintenance

Our S3 flushes happen at either 2 GB or 30 minutes, and given that we partition by date, 
we're creating at least 48 files within a partition alone. This is a lot of small data files, plus 
Iceberg metadata files that S3 has to store. More data files means higher querying latency, and more 
metadata files means more storage costs. In addition, duplicates can appear across runs.

Therefore, it is imperative that we have cron jobs running to dedup, compact, and delete post-compaction artifacts. 

![maintenance system design]({{ "/assets/images/maintenance-architecture.png" | relative_url }})

## Database Design

Our database is not one piece of software. It's five layers stacked on object storage,
each one there because the layer beneath it doesn't do the job alone: S3 holds bytes,
Parquet gives those bytes a shape, Iceberg turns a pile of files into a table, Glue makes
that table addressable by name, and Athena answers questions about it.

![database visual]({{ "/assets/images/database-layers.png" | relative_url }})

### The Bytes: S3

Object storage is the cheap, durable, effectively unbounded floor, which is what we need at
terabyte scale. 

### The File Format: Parquet

**Our users want a lot of rows and only a few columns.** For example, "Give me the last
10000 posts starting from August 23rd 2026" or "Which 10 posts got the most likes this week?"

**OLAP vs OLTP** One way we think about the difference is that 
OLTP is about getting a lot of columns on a few rows, whereas OLAP is about getting a lot of rows
on a few columns. Our users are therefore mainly making OLAP queries. 

Columnar layout is what makes that pattern cheap. If the main query is an OLAP query, we
don't want to pull down whole posts just to read one column of each. Each page in a columnar
file holds contiguous entries from a single column rather than whole rows, so a reader can
fetch only the columns a query actually asks for. It is the difference between dragging back 
every byte we have stored and pulling a handful of byte ranges.

### The Table Format: Iceberg

#### Why Tables

File names alone in S3 have no organization. Nothing can query all posts on August 3rd, 2026
without having to scan all the files in S3. Having a table allows us to partition our files,
optimizing query speeds.

#### Why Iceberg

Plain Glue tables with partition projection over Parquet would have given us the names, the schemas,
and pruning without a metastore lookup for every partition. That covers the first write. 
What it doesn't cover is everything that happens afterward, and a firehose means there is always an afterward.

**Deduplication support.** Iceberg natively provides features like merge-on-read for us. This
allows us to have clean data at query time, and also makes deletes cheap. We're able to easily set up weekly cron jobs for 
this dedup. A tradeoff we're allowing here is that until the cron job runs, we are prone to having a few duplicates, though 
this is rare enough for us to accept it. 

**Compaction is only safe if commits are atomic.** Streaming ingest produces small files that we eventually
want to compact in order to prevent degrading query speeds. That means rewriting files underneath live readers,
but on a Hive table there is no atomic way to swap the old files for the new ones. With Iceberg's snapshot system, 
we're able to atomically switch versions from pre-compaction to post-compaction. 

### The Catalog: Glue

Iceberg describes a table in its own metadata file system, but something still has to know which
metadata file is current, and what `posts` even refers to. That's the catalog: Glue holds
the pointer to the table's current metadata file, which allows us to atomically switch pointers from 
one version to the next. 

### The Engine: Athena

Athena is the engine our users actually query, and it natively understands Iceberg's features, like merge-on-read. 
It's also the maintenance engine, as dedup, compaction, and cleanup all run as Athena statements. 

## DevOps & Observability

The ingester exports OpenTelemetry metrics and logs to Grafana Cloud, where we built a dashboard
that answers the most important questions about our app's health:

**Is it running?** A liveness indicator on the Jetstream socket, and the position of the
resume cursor. We need to make sure at all times that our cursor is less than 72 hours before the present,
as the Bluesky Jetstream only guarantees 72 hours of data freshness. 

**Did we lose anything?** Dead letters and dropped rows. Whenever something is sent into our deadletter queue,
we need to know so that we can manually deal with it. If we have dropped rows, we know that both the regular write
path and the dead letter path failed, so it needs special attention.

**What's actually arriving?** Piecharts and breakdowns of the amount of data that's arriving into our system. We collect
posts, reposts, likes, and follows, so we break down how much of each data is coming into our system per buffer flush, and also for the last 24 hours. 

![Our Grafana dashboard for the ingester]({{ "/assets/images/grafana-dashboard.png" | relative_url }})

## What I Learned
**1. The importance of Experiments** There were many times where we would come up with possible solutions to a problem, but think, 
"I don't know what the correct answer is, so I'm going to test this." This was a mixture of load testing, 
latency experiments, and proof of concepts. I found that this ended up being one of the biggest gains in 
using AI, as the slop code that it wrote for me was a one-off anyway (in fact, we have more code for experiments
than we do the actual app!), but got me the information I needed to make my decisions. 

**2. Don't polish before presenting** I spent months refining the first approach where we used AppView, 
making sure every part of the build was clean and scalable. Then when we presented our app to our users, 
we just received feedback that they wanted a lot more data, and we realized our V1 approach wasn't going to work. 

I easily could have just seeded in some random data and shown an interface to users within a week
and iterated off of that (though I don't regret picking up on patterns learned from building out the V1).
In the future, I would definitely try to get an MVP out to users at all times before even thinking about trying to build polished software. 