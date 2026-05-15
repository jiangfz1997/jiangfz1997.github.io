---
title: "RAG Chatbot over Internal Documents"
description: "A retrieval-augmented generation system that lets teams query their internal knowledge base in plain English."
date: 2026-03-10
tags: ["Python", "LangChain", "OpenAI", "FastAPI", "PostgreSQL", "pgvector"]
github: "https://github.com/jiangfz1997"
featured: true
status: completed
---

## Background

My team had a sprawling internal wiki — hundreds of Confluence pages that nobody read because search was terrible. The standard response to "how does X work?" was to ping a senior engineer, which didn't scale. I built this to fix that.

## What it does

Users type questions in plain English; the system returns answers grounded in the actual documentation, with source links. Unlike a plain LLM, it won't hallucinate — if the answer isn't in the docs, it says so.

## How it works

**Ingestion pipeline**: Confluence pages are fetched via API, chunked with overlap using a sliding window strategy, embedded with `text-embedding-ada-002`, and stored in PostgreSQL with `pgvector`.

**Retrieval**: Hybrid search — dense vector similarity (cosine) plus sparse BM25 keyword matching. Results are re-ranked with a cross-encoder before passing to the LLM. Hybrid consistently outperformed pure vector search on my eval set by ~12% recall@5.

**Generation**: GPT-4o with a system prompt that constrains it to cite sources. Streaming via Server-Sent Events so the UI feels fast.

## The hard parts

**Chunking strategy mattered more than I expected.** Semantic chunking (splitting on topic boundaries) beat fixed-size chunks on long technical docs, but was slower to index. I ended up with a two-pass approach: fixed chunks for ingestion speed, semantic re-chunking for the top-k results at query time.

**Evaluation without labels.** I built a synthetic Q&A set by prompting GPT-4 to generate questions from each source chunk, then measured whether retrieval recovered the correct chunk. This gave me a reproducible regression metric without needing human annotators.

## Results

- Reduced "ping a senior for docs questions" Slack messages by ~60% in the first month
- p95 query latency: 2.1s (including streaming start)
- Recall@5 on synthetic eval set: 0.89

## What I'd do differently

The re-ranking step adds ~400ms of latency and the quality gain is marginal for short factual questions. I'd gate it on query complexity next time — only re-rank when the query looks like it needs multi-hop reasoning.
