---
title: "Evaluating RAG Systems Without Human Labels"
description: "Approaches to automated evaluation of retrieval-augmented generation pipelines when you don't have a golden dataset."
date: 2026-04-15
tags: ["LLM", "RAG", "evaluation"]
---

Evaluating RAG systems is hard. You need to assess at least three things independently:

1. **Retrieval quality** — did we fetch the right context?
2. **Faithfulness** — does the answer stay grounded in the retrieved context?
3. **Answer relevance** — does the answer actually address the question?

The problem: building a labeled evaluation set takes weeks and goes stale every time your data changes.

## LLM-as-judge

The most practical approach I've found for teams without evaluation budgets is using a separate LLM call to score the output. The key insight is to decompose the judgment:

```python
FAITHFULNESS_PROMPT = """
Given the context below and the answer, rate whether the answer contains
only information supported by the context.

Context: {context}
Answer: {answer}

Respond with a score from 0 to 1 and a one-sentence reason.
"""
```

This works surprisingly well for catching hallucinations. The failure mode is that the judge shares the same biases as the generator — so a model that confidently makes things up will often also confidently rate itself as faithful.

## Synthetic question generation

For retrieval evaluation, generate questions from your corpus and check if retrieval recovers the source chunk:

```python
# For each document chunk, ask the LLM:
# "What question does this passage answer?"
# Then run those questions through retrieval and measure recall@k.
```

This gives you a fast regression signal: if a code change drops recall@5 from 0.87 to 0.74, something is wrong before you deploy.

## What I haven't solved

Latency vs. quality tradeoffs are still mostly vibes. And evaluating multi-turn conversations is a different problem entirely. More on that later.
