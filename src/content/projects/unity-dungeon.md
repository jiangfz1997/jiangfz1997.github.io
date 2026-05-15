---
title: "Dungeon Crawler — Unity WebGL"
description: "A procedurally generated dungeon crawler built in Unity, playable in the browser."
date: 2025-11-20
tags: ["Unity", "C#", "WebGL", "Procedural Generation"]
featured: true
status: completed
---

## Why I built this

I wanted to understand procedural generation from first principles — not use a library, but actually implement the algorithms. A dungeon crawler was the right scope: small enough to finish, complex enough to be interesting.

## Core systems

**Map generation** uses a BSP (Binary Space Partitioning) tree. The dungeon floor is recursively split into rooms, then corridors connect adjacent siblings. This guarantees full connectivity without needing a flood-fill check.

**Enemy AI** runs on a simple behaviour tree: patrol → alert → chase → attack. Each state has a clear exit condition. I deliberately avoided NavMesh in favour of A* on the grid — overkill for a dungeon, but I wanted to understand the algorithm properly.

**Combat** is turn-based on a shared initiative queue. Every actor (player + enemies) has a speed stat that determines when their next turn fires. This makes fast enemies feel genuinely threatening without needing real-time input handling.

## Technical notes

**WebGL build size**: first build was 48MB uncompressed. After enabling Brotli compression and stripping unused Unity modules, it's 11MB — small enough to host on GitHub Pages without hitting file size limits.

**Performance**: the main thread bottleneck was map generation on level load. Moved it to a coroutine with a yield per BSP split — imperceptible to the player, no frame drops.

## Play it

The game is embedded in the demos section. Use arrow keys or WASD to move, Space to attack.
