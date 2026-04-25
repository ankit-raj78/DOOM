# 15 — Suggested study exercises

Exercises are graded by depth, not difficulty. Reading-only tasks come first;
hands-on modifications come later. Treat the line numbers as anchors — the
question is "do you understand what is happening here", not "can you reach
the file".

## Reading exercises

1. **Trace one tic, end to end.** Starting at
   [d_main.c:354 D_DoomLoop](../linuxdoom-1.10/d_main.c#L354), follow
   exactly one game tic from `I_StartFrame` through `G_Ticker` →
   `P_Ticker` → `P_PlayerThink` → `P_RunThinkers` → display. Write the call
   chain (no edits). Identify the moment the simulation transitions from
   "reading inputs" to "writing world state."

2. **Find the desync detector.** Locate where `ticcmd_t.consistancy` is
   *written* and where it is *checked*. Why does the value have to be
   non-trivially derived from world state, not, say, a counter?

3. **Map every `gameaction_t` value to its handler.** For each value of
   `gameaction_t`, find the `G_Do*` function it dispatches to. Why is this
   an enum + switch rather than a function-pointer table? (Hint: think
   about save-game format stability.)

4. **Pick one monster.** Open [info.c](../linuxdoom-1.10/info.c) and find
   one monster's state graph (e.g. `MT_POSSESSED`). Draw it as a state
   diagram. Mark every state that fires an action, what the action does,
   and where the graph terminates.

5. **Visplane mathematics.** Study `R_FindPlane` in
   [r_plane.c](../linuxdoom-1.10/r_plane.c). Explain in two sentences why a
   floor that wraps an L-shaped room can use one or two visplanes
   depending on view angle.

6. **Carmack's regret.** Re-read [README.TXT](../README.TXT) where he
   regrets not using the BSP for movement and line-of-sight. Open
   [p_sight.c](../linuxdoom-1.10/p_sight.c) — what does the line-of-sight
   code do today? Sketch how it would change if it used the BSP instead.

## Architecture exercises

7. **Module boundaries.** For each pair of subsystems below, document the
   one (or two) header files that mediate the contract:
   - `R_*` ↔ `P_*`
   - `S_*` ↔ `I_*`
   - `G_*` ↔ `WI_*`
   - `D_*` ↔ `D_net`/`I_net`

   Now grep for cross-prefix function calls (e.g. `R_` symbols inside `P_`
   files). Are any of them violations of the "intended" layer in
   [01](01_module_map.md)?

8. **Determinism audit.** List every place in the source where a
   non-deterministic input enters the simulation. Be exhaustive: clocks,
   `rand()`, `getenv`, file I/O, network. For each, decide whether it
   touches `P_Ticker` or only the presentation layer. What would break if
   any of them touched `P_Ticker`?

9. **The role of `Z_ChangeTag`.** Find every call site. Why does
   `W_CacheLumpNum` *change* the tag of an already-cached lump rather than
   reload it? Trace through the implication for memory pressure scenarios.

10. **Two indices.** Document a query that the BSP can answer in O(log n)
    but the blockmap cannot (or vice versa). Identify code in `P_*` that
    uses the blockmap and explain why a BSP query would fail there.

## Refactoring/design exercises

11. **Add a fifth player.** Identify everything that would need to change to
    raise `MAXPLAYERS` from 4 to 5. Don't do it; just enumerate.
    (Hint: it is more than the literal value. Look at frags arrays, save
    game format, network packet layout, intermission tally widths.)

12. **Replace fixed-point with float.** Sketch the migration plan.
    Identify the three or four files that bear the cost.

13. **Modernise the renderer pipeline.** Per Carmack's hint, redesign
    `R_RenderPlayerView` as a single front-to-back BSP walk that emits
    fragments, then a single back-to-front draw. Where do `visplane_t`s go
    in this redesign? Where does sprite clipping happen?

14. **Plug-in transports.** Replace the implicit linking of `i_net` with a
    runtime-selectable transport interface. Specify the function-pointer
    vtable. Is the cost justified?

15. **Save-game forward compatibility.** Read
    [p_saveg.c](../linuxdoom-1.10/p_saveg.c). Design a versioned save
    format that allows future engine changes to load old saves. What
    invariants does the current format implicitly assume?

## Implementation exercises (if you have a compiler set up)

16. **Add `-recordcrc`.** Augment `M_Random` with a per-tic running CRC
    and dump it to a file per `-recordcrc` flag. Replay a demo and verify
    the CRC matches tic-for-tic. Use it to find a non-determinism bug
    (you may need to introduce one first by misusing `rand()` in a
    thinker).

17. **Render-side instrumentation.** Add per-phase timers around
    `R_RenderBSPNode`, `R_DrawPlanes`, `R_DrawMasked`. Profile a
    representative level and report the breakdown. Where does the time
    actually go?

18. **Visplane overflow recovery.** Replace the static `visplanes[]` array
    with a dynamic one. Quantify the cost (allocation pattern, pointer
    invalidation if any). What did you have to change in `R_DrawPlanes`?

19. **Threadable AI.** Sketch a refactor that allowed `P_RunThinkers` to
    run on a worker thread without changing the deterministic contract.
    What does each monster need to read? Which writes would have to be
    deferred?

20. **Headless mode.** Make a build that runs `P_Ticker` and the demo
    pipeline with `nodrawers = true` and no graphics initialisation. How
    many `I_*` calls become no-ops? What is the resulting LOC of the
    "kernel without presentation"?

## Reading list (if you want to go deeper)

- *Game Engine Black Book: DOOM* — Fabien Sanglard. The book-length
  treatment. Pair it with this guide; the diagrams here let you cross-
  reference what Sanglard describes prose-first.
- *Masters of Doom* — David Kushner. The making-of, not the engine; useful
  for context on *why* certain choices were made under the constraints of
  the time.
- *The Game Engine Architecture* (Jason Gregory). Modern equivalent of
  every concept here, with terminology from contemporary engines.
- The original BSP paper: Fuchs, Kedem & Naylor 1980, "On Visible Surface
  Generation by A Priori Tree Structures". DOOM is the most famous
  practical application.
- *Real-Time Collision Detection* (Christer Ericson) — the modern way to
  do everything `p_map.c` does.
- The DOOM Wiki ([https://doomwiki.org](https://doomwiki.org)) for lump
  format details and source-port compatibility notes.

## Closing

If, after this material, you can read [d_main.c](../linuxdoom-1.10/d_main.c)
end-to-end without a reference open and still know which subsystem each
function call lands in, you have absorbed the architecture. From there,
every modern engine — Source, Unity's DOTS, Bevy ECS, Quake, idTech 4 —
makes sense as a refinement, not a revolution. The pattern is durable.
