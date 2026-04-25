# 08 — Player state

A player is two things: a world-side actor (`mobj_t`, see
[07](07_mobj_thinker.md)) and a player-side struct that carries everything
that is *not* world geometry — inventory, view, HUD state, weapon
animation, cheats, intermission accounting.

Source: [d_player.h](../linuxdoom-1.10/d_player.h),
[p_user.c](../linuxdoom-1.10/p_user.c),
[p_pspr.c](../linuxdoom-1.10/p_pspr.c) (player sprites = first-person weapon),
[p_inter.c](../linuxdoom-1.10/p_inter.c) (pickup/damage).

## Class diagram

```mermaid
classDiagram
    class player_t {
        mobj_t* mo
        playerstate_t playerstate
        ticcmd_t cmd

        fixed_t viewz
        fixed_t viewheight
        fixed_t deltaviewheight
        fixed_t bob

        int health
        int armorpoints
        int armortype

        int[NUMPOWERS] powers
        bool[NUMCARDS] cards
        bool backpack
        int[MAXPLAYERS] frags

        weapontype_t readyweapon
        weapontype_t pendingweapon
        bool[NUMWEAPONS] weaponowned
        int[NUMAMMO] ammo
        int[NUMAMMO] maxammo

        int attackdown
        int usedown
        int cheats   "CF_NOCLIP|CF_GODMODE|CF_NOMOMENTUM"
        int refire

        int killcount
        int itemcount
        int secretcount

        char* message
        int damagecount
        int bonuscount
        mobj_t* attacker
        int extralight
        int fixedcolormap
        int colormap
        pspdef_t[NUMPSPRITES] psprites
        bool didsecret
    }

    class playerstate_t {
        <<enumeration>>
        PST_LIVE
        PST_DEAD
        PST_REBORN
    }

    class cheat_t {
        <<enumeration>>
        CF_NOCLIP    = 1
        CF_GODMODE   = 2
        CF_NOMOMENTUM= 4
    }

    class pspdef_t {
        state_t* state
        int tics
        fixed_t sx, sy
    }

    class mobj_t

    player_t --> playerstate_t
    player_t "1" --> "1" mobj_t : mo (NULL if dead/reborn)
    player_t --> "n" pspdef_t  : psprites
    mobj_t --> player_t : back-pointer
```

## Player state machine

```mermaid
stateDiagram-v2
    [*] --> PST_REBORN
    PST_REBORN --> PST_LIVE : G_DoReborn / spawn mobj
    PST_LIVE --> PST_DEAD : health <= 0
    PST_DEAD --> PST_REBORN : fire pressed after death
    PST_LIVE --> PST_LIVE : per tic via P_PlayerThink
```

Reborn handling is a good example of "one place that owns lifecycle":
`G_PlayerReborn` clones the persistent stats (frags, kill counts) into the
new player object and zeroes everything else.

## Per-tic player simulation

```mermaid
flowchart TD
    PT[P_Ticker] --> Loop[for each playeringame]
    Loop --> PPT[P_PlayerThink]
    PPT --> Move[P_MovePlayer<br/>apply forwardmove, sidemove,<br/>angleturn from cmd]
    PPT --> Use{BT_USE pressed?}
    Use -- yes --> PUseLines[P_UseLines<br/>BSP-walk in front to find usable linedef]
    PPT --> Atk{BT_ATTACK?}
    Atk -- yes --> Pspr[P_FireWeapon<br/>set psprite state]
    PPT --> Bob[P_CalcHeight<br/>update viewz with bob/squat]
    PPT --> Pwr[update powers timers,<br/>damagecount/bonuscount fade]
    PPT --> MoveP[P_MovePsprites<br/>tick weapon animation]
    PPT --> Death{health<=0?}
    Death -- yes --> PD[P_DeathThink]
```

`P_MovePlayer` is interesting because it is the bridge from `ticcmd_t`
deltas to `mobj_t` `momx`/`momy`/`angle`. Movement is then resolved by the
shared `P_TryMove` code (used by every walking mobj) in
[p_map.c](../linuxdoom-1.10/p_map.c).

## First-person weapon as a tiny state machine

`pspdef_t[NUMPSPRITES]` is a 2-slot mini state machine for the weapon and
muzzle flash sprites. It uses the same `state_t` infrastructure as monsters
(see [07](07_mobj_thinker.md)) — the weapon's "ready / fire / flash /
reload" frames are just another walk through the global state table. Action
callbacks like `A_FirePistol`, `A_Punch`, `A_FireBFG` live in
[p_pspr.c](../linuxdoom-1.10/p_pspr.c).

## Items and pickups

```mermaid
sequenceDiagram
    participant Mobj as monster/player mobj
    participant Map as P_TryMove
    participant Touch as P_TouchSpecialThing
    participant Player as player_t

    Mobj->>Map: P_TryMove (each tic)
    Map->>Map: blockmap iteration finds touching things
    Map->>Touch: PIT_CheckThing on MF_SPECIAL
    Touch->>Player: bump ammo / health / armor / weapon / power
    Touch->>Mobj: P_RemoveMobj(special)<br/>(or queue for respawn in deathmatch)
```

Caps and limits are encoded in the `mobjinfo_t` table entries and in
constants in [d_items.c](../linuxdoom-1.10/d_items.c) /
[p_inter.c](../linuxdoom-1.10/p_inter.c).

## Cheats

`cheats` is a bitfield. `IDDQD` toggles `CF_GODMODE`, `IDCLIP` toggles
`CF_NOCLIP`. The string-matching state machine is in
[m_cheat.c](../linuxdoom-1.10/m_cheat.c). It is a cute exercise in writing a
ring buffer that recognises a small fixed alphabet — read it as an example
of "the simplest thing that works."

## Intermission stats

After each level the per-player counts (`killcount`, `itemcount`,
`secretcount`, `frags[]`) are copied into a `wbplayerstruct_t` and passed to
`WI_Start` ([wi_stuff.c](../linuxdoom-1.10/wi_stuff.c)). The intermission
screen is, architecturally, a *separate* gamestate (`GS_INTERMISSION`) with
its own ticker and drawer — see [12](12_game_state.md).

> Read next: [09 — Renderer (BSP traversal pipeline)](09_renderer.md).
