---
title: "Resume Tailoring Assistant"
description: "An LLM-powered system that parses your resume, analyzes job descriptions, and generates a tailored draft through a multi-agent pipeline — with a chat assistant to refine the result."
date: 2026-03-01
tags: ["Python", "FastAPI", "LangGraph", "LangChain", "Vue 3", "PostgreSQL", "Google Gemini"]
github: "https://github.com/jiangfz1997/Resume_helper"
featured: true
status: completed
---

## Background

Tailoring a resume for every job application is tedious but actually matters — a generic resume rarely passes keyword filters or reads well to a hiring manager. I built this to automate the mechanical parts (keyword coverage, formatting consistency, field completeness) while keeping the human in the loop for anything that requires judgment.

## What it does

You upload your resume as a PDF and paste a job description. The system extracts your experience into a structured profile, analyzes the JD for hard requirements, core skills, and preferred qualifications, then generates a tailored draft that preserves your actual experience while improving keyword coverage. A diagnostic panel audits the result at three tiers, and a streaming chat assistant lets you refine specific sections.

## Architecture

The system is organized as four distinct multi-agent workflows, all orchestrated with LangGraph:

**Profile Import:** A two-phase pipeline (parse → enrich) uses Flash Lite for extraction and a rule-based enricher to normalize structure. Two-phase design avoids hallucination creep — the enricher only fills gaps the parser leaves, it doesn't rewrite.

**Resume Generation:** A three-agent pipeline — JD analyzer → skill matcher → resume tailor — runs sequentially in a LangGraph state machine. Each agent has a single responsibility; the skill match report is passed explicitly to the tailor so it doesn't need to re-derive relevance.

**Chat Assistant:** An intent router (Flash Lite, structured output) classifies the user's message and dispatches to one of four handlers: `local_patch`, `keyword_inject`, `full_diagnose`, or `question`. This keeps the response sharp — a "fix this bullet" request doesn't trigger a full re-generation.

**Copilot Panel:** Three-tier audit running in parallel: rule-based field checks (tier 1), LLM macro analysis of overall positioning (tier 2), and per-bullet micro-validation (tier 3). Tier 1 is instant and catches obvious gaps; tiers 2 and 3 are async.

## Tech stack

- **Backend:** Python 3.11, FastAPI, SQLAlchemy (async), PostgreSQL
- **LLM orchestration:** LangGraph, LangChain Core
- **LLM provider:** Google Gemini Flash / Flash Lite
- **PDF:** PyMuPDF for parsing, Playwright (headless Chromium) for export
- **Frontend:** Vue 3, Vite, Naive UI, TypeScript
- **Auth:** JWT + bcrypt

## The hard parts

**Keyword scoring without false precision.** Rule-based coverage metrics sound simple but the edge cases add up — stemming mismatches, synonyms ("managed" vs "led"), acronyms ("ML" vs "machine learning"). I kept the scorer intentionally simple and surfaced low-confidence matches as warnings rather than hiding them.

**PDF export fidelity.** Generating PDFs from HTML via Playwright means the resume looks exactly like the preview, but Playwright adds ~1.5s cold-start latency. Acceptable for an async export flow, but I'd pre-warm the browser process in production.

**Agent configuration without code changes.** All model assignments, temperatures, and token limits live in `agents.yaml`. Swapping a model or tuning a temperature for one agent doesn't require touching any Python — useful when iterating on quality.

## What I'd do differently

The intent router adds a round-trip before the actual chat response. For short Q&A messages it's unnecessary overhead — I'd classify intent client-side for simple cases and only call the router for ambiguous ones.
