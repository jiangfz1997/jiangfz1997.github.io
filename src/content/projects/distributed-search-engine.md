---
title: "Distributed Search Engine"
description: "A full-stack distributed search engine over Simple English Wikipedia, with a MapReduce indexing pipeline, BSP PageRank computation, and hybrid BM25 + PageRank ranking."
date: 2025-12-01
tags: ["Python", "PostgreSQL", "Redis", "Docker", "FastAPI", "MapReduce", "Streamlit"]
github: "https://github.com/jiangfz1997/distributed_searching_engine"
featured: true
status: completed
---

## Overview

A distributed search engine built from scratch over the Simple English Wikipedia corpus. The architecture separates offline data processing from online query serving: a MapReduce-style pipeline builds the inverted index, a BSP graph engine computes PageRank, and a FastAPI backend serves hybrid-ranked results to a Streamlit frontend.

The goal was to build something that actually scales horizontally — adding more indexing or PageRank workers speeds things up without changing any other part of the system.

## Architecture

The system runs entirely in Docker. Each component is a container; Redis and PostgreSQL are the only shared state.

**Offline pipeline (triggered by `run_full_pipeline.py`):**
1. **Ingestion** — parse raw Wikipedia XML, strip markup with `mwparserfromhell`, tokenize and stem with NLTK, emit JSONL.
2. **Indexing (MapReduce)** — Mapper containers emit `(term, doc_id, tf)` pairs; Reducer containers aggregate into an inverted index stored in PostgreSQL JSONB.
3. **PageRank (BSP)** — a Controller container manages iteration state in Redis; Worker containers pull graph partitions, compute rank updates, and push results back. Converges in ~30 iterations.
4. **Export** — persist final metadata and PageRank scores to PostgreSQL for the serving layer.

**Online serving:**
- FastAPI backend retrieves candidates from the inverted index, scores them using BM25 (term relevance) combined with PageRank (authority), and returns ranked results.
- Streamlit frontend for search UI.

## Why BSP for PageRank

Bulk Synchronous Parallel enforces a barrier between iterations: all workers finish their current round before any worker starts the next. This makes convergence detection trivial (the controller just checks whether any delta exceeded the threshold after each barrier) and avoids the consistency problems you'd get with fully async updates on a shared graph.

Redis works naturally here — workers use it as both a task queue (pull your partition) and a result store (push your deltas), and the controller blocks on a counter key until all workers check in.

## Results

- Full Wikipedia pipeline (~40 min on a laptop): indexes the entire Simple English Wikipedia corpus
- Toy dataset (5000 pages, ~3 min): full pipeline including PageRank convergence
- Horizontal scaling: indexing throughput scales linearly with mapper count

## What I'd do differently

PostgreSQL JSONB for the inverted index was convenient but not optimal — a dedicated search store like Elasticsearch or even a custom on-disk format would give better query latency at scale. JSONB works fine for a few million postings but the GIN index overhead adds up. For a real deployment I'd keep Postgres for metadata only and move the index to something purpose-built.
