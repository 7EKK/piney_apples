---
number: 409
title: Nothing piles up while Kite idles in a town: every town on every volume draws the same frame after 12,000 frames as after 3000, and Carmina Gadelica after 120,000
date: 2026-10-14
area: render, world, test
files: crates/piney-game/src/session/tests/idle_town.rs, crates/piney-game/src/session.rs
---

# 409. Nothing piles up while Kite idles in a town: every town on every volume draws the same frame after 12,000 frames as after 3000, and Carmina Gadelica after 120,000

Issue #65's panic ([[408]]) first read as a frame grown past the vertex
buffer's limit: something in Carmina Gadelica adding draws every frame
and never freeing them. [[408]] showed the message means a lost device
instead. This entry measures the town anyway, and nothing grows.

## How it was measured

A probe stood Kite at a town's gate with the pad untouched and stepped
the session. Every frame was rendered on a headless `Gs`, and every few
thousand frames it logged:

- the frame's commands, prims and their vertices, models and their
  mmats, frame-buffer copies and shadow packets;
- the GS's vertices (`Gs::vertices`), its textures and pipelines;
- the process's resident memory and GPU memory (`nvidia-smi`).

## What it found

- **Every town, every volume.** Towns 0-4 on Infection, Mutation,
  Outbreak and Quarantine were each idled for 12,000 frames. From frame
  6000 on, each town's vertices on each volume stay within a few dozen
  of one value, but for Carmina Gadelica's airship. The ranges over the
  four volumes, from frame 3000:

  | town | vertices a frame |
  | --- | --- |
  | Mac Anu | 168,306-169,296 |
  | Dun Loireag | 207,918-218,622 |
  | Carmina Gadelica | 59,934-72,978 |
  | Fort Ouph | 147,087-148,053 |
  | Lia Fail | 177,072-315,432 (Infection's is the larger) |

  The textures and pipelines are flat too. The rises and falls in
  Carmina Gadelica are the airship (on screen or not) and its puffs (40
  to 66 prims).
- **Carmina Gadelica, Mutation, 120,000 frames** (33 minutes of play):
  59,934 to 72,054 vertices from start to end. GPU memory stays at
  210-212 MiB.
  - Without the GS, resident memory holds at 398,916 kB from frame
    20,000 to 120,000, so the game side keeps nothing.
  - With the GS it rises from 434 MB to 472 MB by frame 100,000 and
    then holds. That is the allocator and the driver settling, not a
    growing list.
- **The town at its fullest.** Three grown Grunties in each pen, the
  race's mail read and ITEM COMPLETE's status set (the Event NPC
  standing), in towns 1-4 on Mutation for 30,000 frames each: flat.
- **The story in Carmina Gadelica.** Mutation's event 115 start, whose
  first block is in the town, idled 40,000 frames: 57,864 to 69,990
  vertices, flat.

## Checked

- **`an_idle_town_draws_no_more_as_time_goes_on`** (piney-game) logs in
  to Carmina Gadelica on Mutation, Outbreak and Quarantine. The pens
  are full, the race's mail read and the Event NPC there. After 600
  frames of set-up it idles twice for 3000 frames. The second stretch
  may draw no more than the first: 10 commands and 2 % of the vertices
  of slack for the airship's random puffs. One sprite more a frame would
  be 3000 commands. Measured: Mutation (135, 78,456) then (135, 78,456);
  Outbreak (136, 79,002) then (135, 78,996); Quarantine (136, 79,386)
  then (137, 79,392).
- piney-game's suite (four threads), clippy, fmt, `cairns check` and
  `tools/docs.py check` pass.

**Still unknown:** nothing. What lost the player's device is [[408]]'s
question, and it needs their machine.
