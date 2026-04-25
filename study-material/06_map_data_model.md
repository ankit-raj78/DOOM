# 06 — Map runtime data model (BSP geometry)

When a level is loaded, [p_setup.c](../linuxdoom-1.10/p_setup.c) parses the
nine geometry lumps (`THINGS`, `LINEDEFS`, `SIDEDEFS`, `VERTEXES`, `SEGS`,
`SSECTORS`, `NODES`, `SECTORS`, `REJECT`, `BLOCKMAP`) and produces the
runtime structures defined in [r_defs.h](../linuxdoom-1.10/r_defs.h). The
runtime types are nearly isomorphic to the on-disk types — they substitute
absolute pointers for indices and add `validcount`, `bbox`, and `specialdata`
fields used during simulation and rendering.

This subsystem is shared between play (`P_*`) and refresh (`R_*`), which is
why its types live in `r_defs.h` and not `p_local.h`.

## Class diagram of the runtime map

```mermaid
classDiagram
    class vertex_t { fixed_t x; fixed_t y }

    class sector_t {
        fixed_t floorheight
        fixed_t ceilingheight
        short floorpic
        short ceilingpic
        short lightlevel
        short special
        short tag
        int soundtraversed
        mobj_t* soundtarget
        int[4] blockbox
        degenmobj_t soundorg
        int validcount
        mobj_t* thinglist   "list of mobjs in sector"
        void* specialdata   "thinker_t for moving floors etc"
        int linecount
        line_t** lines
    }

    class side_t {
        fixed_t textureoffset
        fixed_t rowoffset
        short toptexture
        short bottomtexture
        short midtexture
        sector_t* sector
    }

    class line_t {
        vertex_t* v1
        vertex_t* v2
        fixed_t dx; fixed_t dy
        short flags        "ML_BLOCKING etc"
        short special      "linedef trigger type"
        short tag
        short[2] sidenum   "[1]=-1 if one-sided"
        fixed_t[4] bbox
        slopetype_t slopetype
        sector_t* frontsector
        sector_t* backsector
        int validcount
        void* specialdata
    }

    class seg_t {
        vertex_t* v1
        vertex_t* v2
        fixed_t offset
        angle_t angle
        side_t* sidedef
        line_t* linedef
        sector_t* frontsector
        sector_t* backsector
    }

    class subsector_t {
        sector_t* sector
        short numlines
        short firstline   "index into segs[]"
    }

    class node_t {
        fixed_t x; fixed_t y
        fixed_t dx; fixed_t dy   "partition line"
        fixed_t[2][4] bbox
        ushort[2] children "high bit = subsector"
    }

    class mapthing_t {
        short x; short y
        short angle
        short type
        short options
    }

    line_t "1" --> "1..2" side_t
    line_t "1" --> "2"   vertex_t
    side_t --> sector_t
    seg_t  --> line_t
    seg_t  --> side_t
    seg_t  --> sector_t : front/back
    subsector_t --> sector_t
    subsector_t --> "n" seg_t : firstline + numlines
    node_t --> "2" subsector_t : (subtree pointer or BSP node)
```

## On-disk vs runtime

```mermaid
flowchart LR
    subgraph WAD["WAD lumps (mapdef_t variants)"]
        mv[mapvertex_t]
        ml[maplinedef_t]
        ms[mapsidedef_t]
        msec[mapsector_t]
        mss[mapsubsector_t]
        mseg[mapseg_t]
        mn[mapnode_t]
        mt[mapthing_t]
    end
    subgraph RT["Runtime (r_defs.h)"]
        v[vertex_t]
        l[line_t]
        s[side_t]
        sec[sector_t]
        ss[subsector_t]
        seg[seg_t]
        n[node_t]
    end
    mv --> v
    ml --> l
    ms --> s
    msec --> sec
    mss --> ss
    mseg --> seg
    mn --> n
    mt -.kept as-is.-> mt2[mapthing_t<br/>still used at runtime]
```

The conversion happens in `P_LoadVertexes`, `P_LoadLineDefs` etc. in
[p_setup.c](../linuxdoom-1.10/p_setup.c). A linedef's index `sidenum[0]` is
turned into `sides + sidenum[0]` during load, etc.

## The BSP tree

A **B**inary **S**pace **P**artitioning tree precomputed by an external tool
(e.g. `bsp`, `nodebuilder`) is shipped inside the WAD. Each internal node
stores a partition line; each leaf is a `subsector_t` that is **convex** in
2D — its segs (line segments) bound a convex polygon called a *subsector*.

```mermaid
flowchart TD
    N0[node 0<br/>partition (x,y) (dx,dy)] -->|right of partition| N1
    N0 -->|left| N2
    N1[node 1] -->|right| SS0(["subsector 5"])
    N1 -->|left| SS1(["subsector 7"])
    N2[node 2] -->|right| SS2(["subsector 12"])
    N2 -->|left| N3
    N3[node 3] --> SS3(["subsector 22"])
    N3 --> SS4(["subsector 26"])
```

The high bit of `children[i]` (`NF_SUBSECTOR = 0x8000`) flags whether the
child is a leaf or an interior node. The renderer walks this tree
front-to-back from the player's position; each subsector is drawn fully
before its sibling on the far side of the partition. This is the algorithm
that made DOOM possible on a 386: zero overdraw on solid walls.

## Two parallel spatial indices

Because Carmack didn't reuse the BSP for collision, the play simulation has
its own spatial index, the **blockmap**.

```mermaid
flowchart LR
    subgraph "Spatial indices"
        BSP[BSP tree<br/>node_t / subsector_t<br/>used by R_RenderBSPNode]
        BM[BLOCKMAP<br/>128x128 grid of linedef lists<br/>used by P_BlockLinesIterator]
        TL[thinglist per sector<br/>used by sound, AI]
    end
```

- **BSP** is queried with `R_PointInSubsector(x,y)` (which is also called
  from `P_*` to find a thing's containing sector at spawn time and after a
  move).
- **BLOCKMAP** is a uniform grid; each cell stores a list of linedefs that
  intersect it. `P_BlockLinesIterator` and `P_BlockThingsIterator` are how
  movement, line-of-sight, and missile collision queries find candidates
  cheaply. See [p_maputl.c](../linuxdoom-1.10/p_maputl.c).
- **REJECT** is a precomputed sector-vs-sector visibility bit-matrix used to
  short-circuit line-of-sight queries that would otherwise walk the BSP.

In the README ([README.TXT](../README.TXT)), Carmack flags this duplication
as a regret: "I used the BSP tree for rendering things, but I didn't realize
at the time that it could also be used for environment testing."

## Map load activity

```mermaid
sequenceDiagram
    participant G as G_DoLoadLevel
    participant P as P_SetupLevel
    participant W as W_*
    participant Z as Z_Malloc / Z_FreeTags
    participant R as R_*

    G->>P: P_SetupLevel(episode, map, skill)
    P->>Z: Z_FreeTags(PU_LEVEL, PU_PURGELEVEL-1)
    Note over P,Z: blow away last level's data<br/>in O(blocks)
    P->>W: W_GetNumForName("ExMy")
    P->>W: W_CacheLumpNum(idx + ML_VERTEXES, PU_LEVEL)
    P-->>P: P_LoadVertexes
    P-->>P: P_LoadSectors
    P-->>P: P_LoadSideDefs
    P-->>P: P_LoadLineDefs
    P-->>P: P_LoadSubsectors
    P-->>P: P_LoadNodes
    P-->>P: P_LoadSegs
    P-->>P: P_LoadBlockMap
    P-->>P: P_LoadReject
    P-->>P: P_GroupLines  (sector->lines back-pointers)
    P-->>P: P_LoadThings + P_SpawnMapThing per entry
    P-->>P: P_SpawnSpecials (animated lines, scrolling textures)
    P->>R: R_PrecacheLevel  (touch flats / textures / sprites)
```

The spatial structure is **inert** — no thinker walks it. It is a frozen
input to the simulation. Only the dynamic state (mobjs, moving sectors)
changes during play.

> Read next: [07 — Actors: mobj_t and the thinker system](07_mobj_thinker.md).
