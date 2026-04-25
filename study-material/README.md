# DOOM 1.10 — Study Material

Graduate-level walk-through of the [DOOM 1.10 source](../linuxdoom-1.10/)
released by id Software in 1997 and licensed under GPLv2 by ZeniMax.

The material is split into 16 short chapters with Mermaid UML and system
design diagrams. Read in order, or jump to a specific subsystem.

## Table of contents

| # | Chapter | Topic |
|---|---------|-------|
| 00 | [Overview](00_overview.md) | What this study material is, system context, scale |
| 01 | [Module map and naming convention](01_module_map.md) | Subsystem prefixes, layered + actual dependency diagrams |
| 02 | [Main loop and tic model](02_game_loop.md) | `D_DoomLoop`, `TryRunTics`, the 35 Hz simulation |
| 03 | [Input pipeline and ticcmds](03_input_and_ticcmd.md) | `event_t`, `ticcmd_t`, responder chain, demos |
| 04 | [Memory: the Zone allocator](04_zone_allocator.md) | `Z_Malloc`, purge tags, weak references |
| 05 | [Asset pipeline: WAD lumps](05_wad_pipeline.md) | WAD format, lumpcache, mod system |
| 06 | [Map runtime data model](06_map_data_model.md) | `vertex/line/side/sector/seg/subsector/node/blockmap` |
| 07 | [Actors: mobj_t and the thinker system](07_mobj_thinker.md) | `thinker_t`, `state_t`, dispatch via function pointers |
| 08 | [Player state](08_player.md) | `player_t`, weapon psprites, pickups, cheats |
| 09 | [Renderer (BSP traversal pipeline)](09_renderer.md) | front-to-back BSP, drawsegs, visplanes, vissprites |
| 10 | [Sound subsystem and external sndserv](10_sound.md) | `S_*`, `I_*`, separate sndserv process |
| 11 | [Networking: lockstep peer-to-peer](11_networking.md) | tic exchange, retransmit, consistency |
| 12 | [Game state machine](12_game_state.md) | `gamestate_t`, `gameaction_t`, modal overlays |
| 13 | [Portability layer: the i_* abstraction](13_portability.md) | ports & adapters, `doomcom_t` |
| 14 | [Cross-cutting concerns and lessons](14_lessons.md) | Determinism, ownership, what aged badly |
| 15 | [Suggested study exercises](15_exercises.md) | Reading, architecture, refactoring, implementation |

## How to read

- **Diagrams are Mermaid.** Most modern markdown viewers (VS Code, GitHub,
  Obsidian, Typora) render them inline. If yours does not, paste the code
  block into [Mermaid Live Editor](https://mermaid.live).
- **Links are real.** Every reference like
  [d_main.c:354](../linuxdoom-1.10/d_main.c#L354) points to the line in
  this checkout. Open them as you read.
- **The 200-line summary is in [chapter 14, section 9](14_lessons.md#9-the-architecture-in-one-paragraph).**
  If you only have ten minutes, read that paragraph and skim chapter 02.

## Prerequisites

You should be comfortable with:

- C (any standard) including pointers, function pointers, unions, and the
  ABI implications of struct layout.
- Discrete event simulation conceptually.
- Basic graphics (rasterisation, perspective, palettes).
- TCP/IP basics and the difference between TCP and UDP.

You do *not* need to be familiar with game programming — this material
introduces the relevant concepts as it goes.

## Source layout reference

```
DOOM-1/
├── linuxdoom-1.10/   # main game source — all of the I_*/D_*/G_*/P_*/R_*/S_*/W_*/Z_* code
├── ipx/              # DOS IPX network driver (separate executable)
├── sersrc/           # DOS serial/modem driver (separate executable)
├── sndserv/          # Linux sound server (separate process)
├── LICENSE.TXT       # GPLv2
├── README.TXT        # John Carmack's release notes — required reading
└── study-material/   # this guide
```
