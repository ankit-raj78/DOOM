# 02 — Main loop and tic model

The architectural choice that defines DOOM's behaviour more than any other is
the **fixed-tic simulation, decoupled from rendering**. Every fact in this
document — multiplayer, demos, save games, save-game determinism, deterministic
AI — is downstream of this one decision.

Source: [d_main.c:354 D_DoomLoop](../linuxdoom-1.10/d_main.c#L354-L407),
[d_net.c TryRunTics](../linuxdoom-1.10/d_net.c#L632-L767),
[g_game.c G_Ticker](../linuxdoom-1.10/g_game.c).

## The two clocks

| Clock     | Variable                | Rate                | Source                                 |
|-----------|-------------------------|---------------------|----------------------------------------|
| Real time | `I_GetTime()` returns tics since process start | 35 Hz nominal    | [i_system.c](../linuxdoom-1.10/i_system.c) (uses `gettimeofday`) |
| Game time | `gametic`               | 35 Hz, but not wall-clock | [g_game.c](../linuxdoom-1.10/g_game.c) |
| Build time| `maketic`               | local input rate    | [d_net.c](../linuxdoom-1.10/d_net.c)   |

**Invariant**: the game cannot advance `gametic` past the lowest peer's
`maketic`. This is what makes it a lockstep simulation. See
[`TryRunTics`](../linuxdoom-1.10/d_net.c#L632) — it spins on
`while (lowtic < gametic/ticdup + counts)`.

`TICRATE = 35` is hard-coded in [doomdef.h:122](../linuxdoom-1.10/doomdef.h#L122).
The rendering loop runs as fast as the host machine allows.

## The main loop, end to end

```mermaid
sequenceDiagram
    participant OS as Host OS
    participant I as I_* (platform)
    participant D as D_DoomLoop
    participant Net as D_net (TryRunTics)
    participant G as G_Ticker
    participant P as P_Ticker
    participant R as R_RenderPlayerView
    participant S as S_UpdateSounds

    OS->>I: kbd / mouse / clock
    loop forever
        D->>I: I_StartFrame()
        D->>Net: TryRunTics()
        Net->>I: I_StartTic()
        Net->>D: D_ProcessEvents()
        Note over Net: Build local ticcmd<br/>Send/receive net packets<br/>Wait until lowtic >= needed
        Net->>G: G_Ticker()  (one per tic)
        G->>P: P_Ticker()
        G-->>G: HU_Ticker / ST_Ticker / WI_Ticker / F_Ticker / AM_Ticker
        D->>S: S_UpdateSounds(player)
        D->>R: D_Display() -> R_RenderPlayerView()
        D->>I: I_UpdateSound() / I_SubmitSound()
        D->>I: I_FinishUpdate()  (page flip)
    end
```

Note that `TryRunTics` may run **zero, one or many** `G_Ticker` calls per
display frame. If the renderer is slow, ticks queue up and the next call
catches the simulation back up; if the renderer is fast, the loop spins in
`I_GetTime` waiting for tic-rate to elapse. This is the classic *"fixed
timestep, variable render"* pattern, formalised much later by Glenn Fiedler
(2004) but already shipped here.

## Activity diagram of one display frame

```mermaid
flowchart TD
    Start([frame start]) --> SF[I_StartFrame]
    SF --> Try[TryRunTics]

    subgraph Try["TryRunTics (d_net.c)"]
        T1[Read OS input<br/>I_StartTic + D_ProcessEvents]
        T2[Build local ticcmd<br/>G_BuildTiccmd]
        T3[Send/recv net packets<br/>HSendPacket / GetPackets]
        T4{lowtic >= needed?}
        T5[run G_Ticker<br/>and increment gametic]
        T1 --> T2 --> T3 --> T4
        T4 -- no --> T3
        T4 -- yes --> T5 --> T4exit([return])
    end

    Try --> Snd[S_UpdateSounds]
    Snd --> Disp[D_Display]
    Disp --> Wipe{wipe?}
    Wipe -- no --> Flip[I_FinishUpdate]
    Wipe -- yes --> WipeLoop[wipe_ScreenWipe + I_FinishUpdate per tic]
    WipeLoop --> Flip
    Flip --> SubSnd[I_UpdateSound/I_SubmitSound]
    SubSnd --> Start
```

## What runs inside G_Ticker

`G_Ticker` is the single dispatcher for everything the simulation does in one
tic. It is *not* a thread or a coroutine — it is a sequential pipeline.
Source: [g_game.c](../linuxdoom-1.10/g_game.c) (function `G_Ticker`).

```mermaid
flowchart LR
    GT[G_Ticker] --> GA{gameaction?}
    GA -- ga_loadlevel --> LL[G_DoLoadLevel]
    GA -- ga_newgame --> NG[G_DoNewGame]
    GA -- ga_loadgame --> LG[G_DoLoadGame]
    GA -- ga_savegame --> SG[G_DoSaveGame]
    GA -- ga_completed --> CO[G_DoCompleted]
    GA -- ga_victory --> VV[G_DoVictory]
    GA -- ga_worlddone --> WD[G_DoWorldDone]
    GA -- ga_nothing --> Cont
    LL & NG & LG & SG & CO & VV & WD --> Cont
    Cont[per-player input apply] --> GS{gamestate?}
    GS -- GS_LEVEL --> P[P_Ticker]
    GS -- GS_INTERMISSION --> WI[WI_Ticker]
    GS -- GS_FINALE --> F[F_Ticker]
    GS -- GS_DEMOSCREEN --> D[D_PageTicker]
    P --> ST[ST_Ticker]
    P --> AM[AM_Ticker]
    P --> HU[HU_Ticker]
    WI --> ST
    F --> ST
    D --> ST
```

Inside `P_Ticker` ([p_tick.c:130](../linuxdoom-1.10/p_tick.c#L130-L158)) the
order is again strict and worth memorising:

```c
for (i=0; i<MAXPLAYERS; i++)
    if (playeringame[i])
        P_PlayerThink(&players[i]);   // apply ticcmds

P_RunThinkers();        // every actor's brain
P_UpdateSpecials();     // animated textures, scrollers
P_RespawnSpecials();    // respawn pickups in deathmatch
leveltime++;
```

## The "single-tic" debug mode

[d_main.c:374-386](../linuxdoom-1.10/d_main.c#L374-L386) shows a useful
shortcut for instrumentation:

```c
if (singletics) {
    I_StartTic();
    D_ProcessEvents();
    G_BuildTiccmd(...);
    M_Ticker();
    G_Ticker();
    gametic++; maketic++;
}
```

This bypasses `TryRunTics` entirely and runs **exactly one tic per render
frame**, decoupling the simulation from real time. It is invaluable for
profiling because it removes both the busy-wait and the catch-up multi-tic
case.

## Lessons

- The *only* place where the game checks the clock is inside `TryRunTics`.
  Everything downstream is in tic-space.
- Code that reads from RNG (`M_Random` in [m_random.c](../linuxdoom-1.10/m_random.c))
  is therefore fully deterministic given a seed and a sequence of `ticcmd_t`s.
  This is what allows DOOM demos (LMP files) to replay perfectly: the demo file
  is just a stream of `ticcmd_t`s.
- A modern variant — for instance Quake 3's `cl_predict` and snapshot system,
  or rollback netcode in fighting games — preserves *exactly* this property.

> Read next: [03 — Input pipeline and ticcmds](03_input_and_ticcmd.md).
