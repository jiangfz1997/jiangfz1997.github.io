---
title: "Runtime Weapon Synthesis via LLM Multi-Agent Pipelines"
description: "A research system that generates novel weapons in a live roguelike game at runtime using a multi-agent LLM pipeline — no pre-defined assets, no gameplay interruption."
date: 2026-02-01
tags: ["Python", "FastAPI","LangChain", "Unity", "C#", "MongoDB", "WebSocket", "LLMs"]
featured: true
status: completed
---

## Background

Most game AI research focuses on teaching AI to *play* games. The game world itself — items, enemies, mechanics — stays hard-coded by humans. This project flips that: instead of AI as a player, AI acts as the game system itself, synthesizing content that was never authored by a human.

The testbed is a 2D top-down roguelike (think Vampire Survivors), where weapons are not selected from a static pool but synthesized on the fly by an LLM pipeline based on player context. The challenge isn't just making the AI creative — it's making it fast enough and reliable enough to work inside a real-time game loop.

*Research project at Western University, directed by Prof. Umair Rehman.*

## The core problem: bridging AI and a game engine

Three hard constraints shaped the architecture:

**The logic translation gap.** If the AI generates free-form text, the engine can't use it. If it generates raw code, you get crashes and security risks. The solution: a library of pre-built atomic "primitives" (think LEGO blocks) that the AI assembles rather than writes. Motion primitives handle movement (`OP_MOVE`, `OP_ARC`, `OP_ROTATE`); effect primitives handle combat logic (`OP_MODIFY_HP`, `OP_APPLY_FORCE`, `OP_TIMER` for DOT/HOT); appearance handles visuals. The AI's entire output is a JSON blueprint — no code generation at all.

**The static client bottleneck.** Unity is pre-compiled, so you can't inject new classes at runtime. The fix: C# Reflection. At startup, the game scans its own assemblies and builds a registry of `[PrimitiveId] → Type` mappings. When an AI-generated JSON blueprint arrives, the factory deserializes it directly into runtime objects using that registry. Adding a new primitive type requires zero changes to the factory — it's picked up automatically.

**Reliability and latency.** LLM generation is non-deterministic and slow. The pipeline needed to guarantee 100% usable output within a fixed time window, which required attacking the problem from three angles: input optimization, output validation, and fallback handling.

## The multi-agent pipeline

The backend runs as a LangChain-orchestrated graph with these key stages:

1. **DB Retrieval + Cache:** Fetches similar existing weapons from MongoDB to give the AI examples. Identical requests are served directly from cache — instant, deterministic.
2. **Designer:** Generates a weapon concept and decides whether to reuse an existing weapon or create a new one.
3. **Parallel factories (on new weapon):** `payload_factory`, `projectile_factory`, `weapon_designer`, and `artist` run concurrently. Splitting the work by domain reduces cognitive load per agent and cuts wall-clock time.
4. **Payload Validator:** Pydantic schema enforcement — not a slow LLM check, a programmatic one. 100% valid JSON in milliseconds.
5. **Weapon Patcher:** If validation fails, a lightweight patcher node fixes specific field-level violations without re-running the full generation.
6. **Power Budget:** Scales weapon stats to match current difficulty using a formula (`Budget = 10 × worldLevel^1.5`), preventing both underpowered and game-breaking weapons purely mathematically — no LLM involved.

## Turning latency into a feature

Mean generation time is ~8 seconds. Rather than hiding this or freezing the game, the UX design leans into it:

- **Phase 1 (Trigger):** Player requests a new weapon; generation starts async. Game continues uninterrupted.
- **Phase 2 (Masking):** Enemy spawn rates spike and "charging" VFX fires. The player is under pressure — they're not waiting, they're fighting.
- **Phase 3 (Payoff):** Weapon arrives and instantly equips. The newly generated weapon is intentionally overpowered to give the player a cathartic release.

The latency becomes the tension arc.

## Results (100-run evaluation)

| Metric | Result |
|---|---|
| Success rate | 100% (100/100 parsed and instantiated) |
| Fallback rate | 1% |
| Mean latency | 8.04s |
| p95 / p99 latency | 10.32s / 13.76s |
| 92nd percentile | within 10s |
| Tokens per generation | ~18.2K |
| Cost per weapon | $0.0169 |
| Unique weapons generated | 59/100 (41 cache hits) |

## What I'd do differently

The artist node is the slowest because it still makes an LLM call to select visual assets. Since the selection space is finite (pre-built sprites), this could be replaced with a simple keyword-to-asset lookup table — faster and cheaper, no quality loss. The LLM adds nothing when the answer is a fixed enumeration.
