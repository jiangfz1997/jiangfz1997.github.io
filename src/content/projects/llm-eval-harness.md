---
title: "LLM Evaluation Harness"
description: "A lightweight framework for running reproducible evals on LLM outputs — no config files, just Python."
date: 2026-01-15
tags: ["Python", "OpenAI", "Pytest", "CLI"]
github: "https://github.com/jiangfz1997"
featured: false
status: in-progress
---

## The problem

Every LLM project eventually needs evaluation, and everyone builds the same scaffolding: load prompts, call the API, compare outputs, report metrics. I kept copy-pasting this between projects.

## What it does

A thin harness that turns a directory of `(input, expected)` YAML files into a structured eval run:

```bash
llm-eval run ./evals/summarization --model gpt-4o --judge gpt-4o-mini
```

Output is a JSON report + a markdown summary you can paste into a PR description. Supports:

- **LLM-as-judge** scoring with configurable rubrics
- **Exact match** and **regex match** for structured outputs
- **Diff view** between two runs (useful for catching regressions before deploying a prompt change)
- **Cost estimation** before running (so you don't accidentally spend $50 on a misconfigured eval)

## Design decisions

**No config files.** Everything is either CLI flags or Python code. YAML for test cases only, never for framework behaviour. Configuration-as-code is easier to version and diff than a `config.yaml` that accumulates cruft.

**Pytest-compatible.** Each eval case is a pytest test under the hood. This means you get free integration with CI, parallel execution, and `--lf` (last failed) reruns.

## Status

Core eval loop is stable and I use it daily. Working on a web UI for browsing eval history across runs.
