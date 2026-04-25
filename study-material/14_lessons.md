# 14 — Cross-cutting concerns and lessons

This chapter pulls themes that are not subsystem-local. Read it after the
others — it makes more sense once you know the moving pieces.

## 1. Determinism is the keystone

Every "interesting" feature — replay/demos, peer-to-peer multiplayer,
save/load — falls out of one design contract:

> The simulation is a deterministic function of `(state_{t}, ticcmd_{t}) →
> state_{t+1}`.

Every architectural choice is consistent with that contract:

- **Fixed-rate `tic`** → simulation does not depend on wall-clock.
- **Custom RNG `M_Random`** ([m_random.c](../linuxdoom-1.10/m_random.c)) → a
  256-byte LUT and an integer index. Indexed by `M_Random()` and seeded
  identically on every peer. Game code uses `M_Random` always; UI code uses
  `M_BiRandom` or similar. Crucially, *no one is allowed to call `rand()`
  during simulation*.
- **Inputs as data** (`ticcmd_t`) → record/replay/network are the same
  pathway.
- **Single ordered list of thinkers** → tick order is reproducible.

The single biggest "modern lesson" is to identify what your domain's
deterministic-pure-function looks like, and protect it. Anything stochastic
or wall-clock-dependent must live outside the kernel. Compare to:

- React: a render is a pure function of `(props, state)`.
- Redux: state evolves only via dispatched actions.
- Event-sourced systems: state is a fold of an append-only event log.

DOOM is event-sourced. The event log is the demo file.

## 2. Memory ownership is encoded in *tags*, not types

The Z_zone allocator (see [04](04_zone_allocator.md)) lets a caller declare
"I want this allocation to die when the level ends" without writing
explicit cleanup code. This is conceptually identical to:

- Region-based allocation (Cyclone, Apache memory pools).
- Java/C# generational collectors with custom roots.
- Rust lifetimes (compile-time encoding of the same idea).
- Arena allocators in modern C/C++ engines.

The DOOM scheme is the simplest version: a one-byte tag. The lesson is
that **lifetime is a first-class design concept**; you should know who owns
each allocation and when it dies. If you can't say in one sentence, you have
a bug in waiting.

## 3. Composition by file prefix is a real namespace

DOOM has no namespaces, classes, or modules. It has filename prefixes.
After 30 years and millions of forks, this convention is still legible.
Why?

- It is **searchable**. `grep '^P_'` is the package listing.
- It is **uniform**. No exceptions.
- It is **enforced by the file boundary**, which is the only structural
  unit C has.

Modern projects with `Foo::Bar::Baz::Quux::Frobnicator` would do well to
remember that a flat 2-letter prefix scaled to 50k LOC of game logic.

## 4. Platform abstractions via compile-time linking

The `I_*` interface (see [13](13_portability.md)) shows an alternative to
runtime polymorphism: declare the port in a header, link exactly one
adapter. No virtual dispatch, no plugin manager, no reflective registry.
This is what `getrandom(2)` does on Linux today, what BoringSSL does for
PRNG providers, what the Linux kernel's `arch/*` tree does.

## 5. Two-pass spatial structures

DOOM keeps two parallel 2D indices because the queries differ
(see [06](06_map_data_model.md)):

- BSP for **render front-to-back** ordering.
- Blockmap for **point-in-region** during play simulation.

Carmack himself flagged this as a regret — he could have reused the BSP for
both. The lesson generalises: every duplicated index is a maintenance bill
and a potential source of skew. Whenever you see two structures indexing
the same data, ask whether you can pick one.

## 6. Cooperative multitasking inside a frame

`R_RenderPlayerView` calls `NetUpdate` between phases (see
[09](09_renderer.md)). The screen-wipe loop does the same. There is no
thread; the renderer voluntarily yields by calling into the network code.
This pattern reappears in modern engines as fiber scheduling and
work-stealing job graphs. The contract is the same: long-running work must
voluntarily check in or it will block other subsystems.

## 7. State machines, everywhere

Mobjs animate via `state_t` graphs. The whole game runs as a `gamestate_t`
state machine. Player life cycle is a `playerstate_t` machine. Game
transitions are a `gameaction_t` deferred command queue. The combined effect
is that **at every level of zoom, you can answer "what state are we in?"**.
Modern UI frameworks (XState, Statecharts) and front-end stores (Redux
Sagas) reach for the same primitive.

## 8. What aged badly

Reading honestly, DOOM has flaws that you should learn to recognise:

- **Fixed-point math in a public type system.** `fixed_t` is a typedef of
  `int`. Mixing units is a real bug source — you will see `*FRACUNIT` and
  `>>FRACBITS` everywhere. A modern engine uses `float` (or wraps `Fixed16`
  in a real type that errors at compile time on mixed ops).
- **Globals.** Practically every interesting variable is a global —
  `gametic`, `players`, `gamestate`, `viewx`, `viewangle`. This is the
  reason multiple-window or split-screen support requires invasive
  surgery.
- **Hardcoded resolution and aspect.** `SCREENWIDTH 320`, `SCREENHEIGHT 200`
  are macros, not parameters. `r_*` is correct only at this size. Source
  ports parameterise these, but the change is large.
- **`(++eventhead)&(MAXEVENTS-1)`** is technically UB in C
  ([d_main.c:153](../linuxdoom-1.10/d_main.c#L153)). Modern compilers warn
  on it and may miscompile in some optimisation modes.
- **Function pointer unions** (`actionf_t`) abuse the language. Only one
  field is ever live; the union exists to avoid casts. A modern equivalent
  uses tagged unions or `std::variant`.
- **Pre-emptive small-buffer assumptions** all over (e.g. `MAXVISPLANES`,
  `MAXDRAWSEGS`). When exceeded, the game crashes. Source ports replaced
  these with growable arrays. The right reflex now: if you write a
  fixed-size buffer you must also write a "full" handler.
- **`I_Error` is the universal panic.** Anything from "WAD not found" to
  "thinker corruption" reaches `I_Error("...")`, which prints and exits.
  No recovery. Worth contrasting with modern game engines that survive
  asset failures.

## 9. The architecture in one paragraph

DOOM is a deterministic 35 Hz simulation kernel (`P_*`) wrapped in a
control plane (`G_*` and `D_*`) that consumes serialisable input
(`ticcmd_t`) drawn from one of three sources (live keyboard, demo file,
network), produces world updates that obey a strict tic order with one
deferred-action queue (`gameaction_t`) for irreversible operations, and is
rendered by a separate front-to-back BSP traversal pipeline (`R_*`) and
sound subsystem (`S_*` + `i_sound` + sndserv) that read the world state
without mutating it. A custom arena allocator (`Z_*`) with purgeable cache
tags backs every dynamic allocation and feeds a WAD-based asset pipeline
(`W_*`) that decouples content from binary. All system calls funnel through
a tight platform port (`I_*`).

That is a lot of value out of 54k lines of C.

> Read next: [15 — Suggested study exercises](15_exercises.md).
