# Quake III Arena — A Deep Reading Companion

> **Companion to [STUDY_GUIDE.md](STUDY_GUIDE.md).** That guide is a *visual map*: C4
> diagrams, Mermaid flowcharts, class diagrams, and "why it matters" callouts that
> orient you in the architecture. **This guide is its narrative complement**: a
> reading order, a textual deep-dive into the subsystems the visual guide
> compresses, hands-on exercises, and material on parts of the codebase the visual
> guide leaves out — the LCC compiler, the offline VIS and radiosity pipeline in
> `q3map`, the adaptive Huffman coder, and the sound subsystem.
>
> If [STUDY_GUIDE.md](STUDY_GUIDE.md) is the lecture, this is the recitation.
> Read them side by side.

---

## 0. How to Use This Guide

This guide assumes a graduate student who is comfortable with C, basic computer
graphics, OS concepts (virtual memory, processes, scheduling), and elementary
information theory. Allow **20–30 hours** of focused source reading. The recommended
flow:

1. Skim [STUDY_GUIDE.md](STUDY_GUIDE.md) sections 0–4 for the big picture (45 min).
2. Read this guide's §1–§4 for context and reading order (1 hour).
3. Walk the source in the order given in §4, returning to either guide as you hit
   unfamiliar territory.
4. Tackle the exercises in §13 — they are the part that makes the reading stick.

A repeated theme: **the engine is divided by an ABI boundary** (the QVM syscall
interface). Knowing which side of the boundary a piece of code lives on is half
the battle.

---

## 1. Historical and Pedagogical Context

Quake III Arena shipped December 1999. The source was released under the GPL on
August 19, 2005. The codebase predates:

- C99 (no `stdint.h`, `_Bool`, designated initializers, mixed declarations)
- Out-of-order superscalar CPUs with branch predictors that "just work"
- GPUs with programmable shaders (the renderer assumes OpenGL 1.1 fixed-function)
- Broadband internet for most consumers — the network code is tuned for 28.8K and
  56K modems
- The dominance of dynamic linking (DLLs are supported, but the *primary*
  extension mechanism is a hand-rolled bytecode VM)
- Multi-core CPUs as the norm (an SMP path exists in the renderer for dual-CPU
  workstations)

The constraints those absences create — *every byte of bandwidth matters, every
cache miss matters, every malloc is suspect, every platform is different* — are
visible everywhere in the source. Modern engineering papers over those constraints
with hardware and abstractions; reading Q3 is an opportunity to see them exposed.

The other thing to internalize: **id wrote this expecting a hostile execution
environment.** Mods are user-supplied code. Maps are user-supplied data. Networks
are adversarial. Files come out of zip archives written by anyone. The defensive
style is intentional, not paranoid.

Read [README.txt](README.txt) — it covers licensing carve-outs (zlib, MD4, JPEG,
ADPCM, BSD libc routines) and the original build instructions.

---

## 2. Repository Layout

```
Quake-III-Arena/
├── code/        Engine + game modules + tool client/server bindings
├── lcc/         Retargetable C compiler used to compile game modules to QVM bytecode
├── q3asm/       Assembler that turns lcc output into QVM bytecode
├── q3map/       Map compiler:  .map → .bsp (BSP, PVS, lightmaps)
├── q3radiant/   The Q3Radiant level editor (Win32 MFC application)
├── common/      Support code shared by Radiant
├── libs/        Third-party libraries (jpeg6, pak/zlib)
├── ui/          Team Arena UI source (separate from code/q3_ui)
├── README.txt
└── COPYING.txt  GPL v2
```

Inside [code/](code/):

| Directory | Role | What lives here |
|---|---|---|
| [code/qcommon/](code/qcommon/) | "common" engine layer | VM, files, network, collision, command system, cvars, memory |
| [code/server/](code/server/) | Authoritative game server | Snapshots, client management, entity world model |
| [code/client/](code/client/) | Game client (engine side) | Connection, snapshot parsing, sound, input, console |
| [code/renderer/](code/renderer/) | OpenGL renderer | Frontend/backend split, shaders, BSP/MD3 rendering |
| [code/game/](code/game/) | **Server game module** (compiled to QVM) | Entity logic, physics, weapons, bots |
| [code/cgame/](code/cgame/) | **Client game module** (compiled to QVM) | HUD, view, prediction, particles |
| [code/q3_ui/](code/q3_ui/) | **UI module** (compiled to QVM) | Menus |
| [code/botlib/](code/botlib/) | Engine-side bot library | AAS pathfinding, chat AI |
| [code/bspc/](code/bspc/) | BSP-to-AAS compiler | Standalone tool that builds bot navigation data |
| [code/jpeg-6/](code/jpeg-6/) | JPEG decoder | Used for textures |
| [code/win32/](code/win32/), [code/unix/](code/unix/), [code/macosx/](code/macosx/) | Per-OS layer | `Sys_*` calls: window, input, threads, files |
| [code/null/](code/null/) | Stub OS layer | Used for the dedicated server build |

The most important conceptual split: **`code/qcommon/`, `code/server/`,
`code/client/`, and `code/renderer/` are the engine** (compiled natively per
platform), while **`code/game/`, `code/cgame/`, and `code/q3_ui/` are *modules***
(compiled by the in-tree LCC and shipped as bytecode). This separation is the
central architectural decision of the engine.

---

## 3. Architectural Overview

```
                ┌────────────────────────────────────────────┐
                │              ENGINE (native code)           │
   keyboard ──▶ │  client ◀──┐         server ◀── network     │
   mouse        │            │           │                    │
                │  renderer  │     CM (collision)             │
                │  sound     │     VM (bytecode interpreter   │
                │            │         + x86 JIT)             │
                │  files / cvars / cmd / common / netchan     │
                └─────┬───────────────────────┬───────────────┘
                      │                       │
                      │   syscall ABI         │   syscall ABI
                      ▼                       ▼
                ┌──────────────┐      ┌──────────────┐
                │  cgame.qvm   │      │   game.qvm   │
                │  (presents)  │      │ (authority)  │
                │              │      │              │
                │  ui.qvm      │      │  bot AI      │
                └──────────────┘      └──────────────┘
                       MODULES (bytecode, sandboxed)
```

What the picture captures:

1. **Two processes-or-not.** Server and client live in the same binary in
   single-player and listen-server modes, in separate binaries when running
   dedicated. The code does not particularly care; they communicate through
   `msg_t` byte buffers either way.
2. **Three modules per side.** Server runs `game.qvm`; client runs `cgame.qvm`
   and `ui.qvm`. Each is sandboxed: no direct syscalls, no direct memory access
   outside its data segment, no direct OpenGL/sound/network — only the
   engine-defined ABI.
3. **Shared physics.** [code/game/bg_pmove.c](code/game/bg_pmove.c) is compiled
   into *both* `game.qvm` (server-authoritative) and `cgame.qvm` (client-side
   prediction). The shared movement code is what makes lag-tolerant FPS possible.
4. **Network is a serialization boundary.** All state delivered to clients goes
   through [code/qcommon/msg.c](code/qcommon/msg.c) delta encoding (§7).

A typical frame on a listen server:

```
SV_Frame(msec):
    ├── SV_CheckTimeouts()
    ├── SV_CalcPings()
    ├── VM_Call(game.qvm, GAME_RUN_FRAME, sv.time)   ─┐
    │       └── G_RunFrame() → G_RunThink() per ent   │  authoritative
    │       └── ClientThink() per client (Pmove)      │  simulation
    │       └── CheckTriggers, CheckTeamLeader, etc.  ─┘
    └── SV_SendClientMessages()                        ─── snapshot delta-encoded

CL_Frame(msec):
    ├── CL_ReadPackets() → CL_ParseSnapshot()          ─── apply server delta
    ├── CL_CreateNewCommands() → poll input            ─── build usercmd_t
    ├── CL_SendCmd()                                   ─── send to server
    ├── SCR_UpdateScreen():
    │     └── VM_Call(cgame.qvm, CG_DRAW_ACTIVE_FRAME) ─── prediction + render
    │           └── CG_PredictPlayerState() (Pmove)
    │           └── CG_AddPacketEntities()
    │           └── R_RenderScene() (frontend → backend)
    └── S_Update()                                     ─── audio mix
```

When server and client are the same binary, both `SV_Frame` and `CL_Frame` are
called from the main loop in [code/qcommon/common.c](code/qcommon/common.c)
(`Com_Frame`).

---

## 4. Suggested Reading Order

Reading top-to-bottom by directory will lose you. Use this order:

| Step | Read | Why |
|---|---|---|
| 1 | [code/game/q_shared.h](code/game/q_shared.h) | Names and types you will see everywhere: `vec3_t`, `playerState_t`, `entityState_t`, `usercmd_t`, `trace_t`, `trajectory_t`. |
| 2 | [code/game/q_math.c](code/game/q_math.c) | Vector math + the famous `Q_rsqrt`. Short, self-contained warm-up. |
| 3 | [code/qcommon/qcommon.h](code/qcommon/qcommon.h) | Engine-internal interfaces. Skim — you will return repeatedly. |
| 4 | [code/qcommon/common.c](code/qcommon/common.c) | `Com_Init`, `Com_Frame`, the Hunk/Zone allocators. |
| 5 | [code/qcommon/cmd.c](code/qcommon/cmd.c), [code/qcommon/cvar.c](code/qcommon/cvar.c) | Console commands and global variables. Tiny, foundational. |
| 6 | [code/qcommon/files.c](code/qcommon/files.c) | The virtual filesystem and `.pk3` overlay. |
| 7 | [code/qcommon/msg.c](code/qcommon/msg.c) | Bit-packed message I/O. Gateway to networking. |
| 8 | [code/qcommon/net_chan.c](code/qcommon/net_chan.c) | Reliable channel over UDP, with fragmentation. |
| 9 | [code/qcommon/huffman.c](code/qcommon/huffman.c) | Adaptive Huffman compression of every packet. |
| 10 | [code/server/sv_snapshot.c](code/server/sv_snapshot.c) | Per-client snapshot building — heart of network play. |
| 11 | [code/client/cl_parse.c](code/client/cl_parse.c) | The other side: parsing and applying snapshots. |
| 12 | [code/qcommon/vm.c](code/qcommon/vm.c), [code/qcommon/vm_local.h](code/qcommon/vm_local.h) | The VM: how modules are loaded and called. |
| 13 | [code/qcommon/vm_interpreted.c](code/qcommon/vm_interpreted.c) | Read this *before* the JIT — it is the executable specification. |
| 14 | [code/qcommon/vm_x86.c](code/qcommon/vm_x86.c) | The x86 JIT compiler. ~1200 lines that emit machine code. |
| 15 | [code/game/g_main.c](code/game/g_main.c), [code/game/g_public.h](code/game/g_public.h) | Module entry points and syscall ABI definition. |
| 16 | [code/game/bg_pmove.c](code/game/bg_pmove.c) | Player movement, the same code on server and client. |
| 17 | [code/cgame/cg_main.c](code/cgame/cg_main.c), [code/cgame/cg_predict.c](code/cgame/cg_predict.c) | Client-side prediction and entity interpolation. |
| 18 | [code/qcommon/cm_trace.c](code/qcommon/cm_trace.c) | Collision detection — BSP traversal, brush clipping. |
| 19 | [code/renderer/tr_main.c](code/renderer/tr_main.c), [code/renderer/tr_backend.c](code/renderer/tr_backend.c) | Frontend/backend split. |
| 20 | [code/renderer/tr_shader.c](code/renderer/tr_shader.c) | The shader script system. |
| 21 | [code/client/snd_dma.c](code/client/snd_dma.c), [code/client/snd_mix.c](code/client/snd_mix.c) | Audio mixer. |
| 22 | [code/botlib/be_aas_main.c](code/botlib/be_aas_main.c) | Bot navigation. Optional. |
| 23 | [q3map/bsp.c](q3map/bsp.c), [q3map/vis.c](q3map/vis.c), [q3map/lightv.c](q3map/lightv.c) | The offline pipeline that makes the runtime cheap. |
| 24 | [lcc/src/main.c](lcc/src/main.c), [lcc/src/dag.c](lcc/src/dag.c), [lcc/src/bytecode.c](lcc/src/bytecode.c) | The retargetable C compiler. Optional but enlightening. |

---

## 5. Foundations: Memory, Strings, Math

### 5.1 The two-tier allocator

The engine exposes two allocators, used for different *kinds of lifetimes*:

- **Zone** (`Z_Malloc` / `Z_Free`) — general-purpose tagged heap. Every allocation
  carries a tag, and `Z_FreeTags(tag)` frees every block with that tag. Good for
  transient or per-subsystem state.
- **Hunk** (`Hunk_Alloc`) — bump allocator over a large pre-allocated region.
  Allocations are *never freed individually*; the hunk has only a "high mark" and
  a "low mark" you can reset to release everything since the last mark. Used for
  level data: BSP, textures, models, shaders. When the level changes, the hunk
  is reset.

Both live in [code/qcommon/common.c](code/qcommon/common.c). Notice how this
maps onto the lifetimes of game state: per-frame allocations go to the zone,
per-level allocations go to the hunk, and the engine never has to chase down
memory at subsystem shutdown because the allocator's structure makes the policy
explicit.

> **Concept.** This is *region-based memory management*, decades before Rust
> borrow checkers and ten years before Apache served HTTP requests with apr
> pools. The point is not to avoid `malloc` for performance; it is to make
> ownership obvious and bugs impossible. Half of "modern" memory-safety
> techniques are reinventions of this.

### 5.2 `vec3_t` and the array-typedef trick

[code/game/q_shared.h](code/game/q_shared.h) defines
`typedef float vec3_t[3];`. Note the consequence: `vec3_t` is an *array*, so it
cannot be returned from a function or assigned with `=`. You will see
`VectorCopy(a, b)` (a macro) instead. This style choice prevents accidental
by-value copies and makes the vector pipeline visible in a profiler.

[code/game/q_math.c:552](code/game/q_math.c#L552) is the famous fast inverse
square root:

```c
float Q_rsqrt( float number )
{
    long i;
    float x2, y;
    const float threehalfs = 1.5F;

    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;                       // evil floating point bit level hacking
    i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
    return y;
}
```

One Newton-Raphson iteration applied to a clever initial guess derived from the
IEEE-754 bit layout of a float. The constant `0x5f3759df` minimizes maximum
relative error of the seed. The aliasing through `long*` is undefined behavior
under modern C and would now produce a strict-aliasing warning, but on
1999-vintage compilers it produced exactly the bit reinterpretation the author
wanted. The "what the fuck?" comment at line 561 is original.

### 5.3 The portable C subset

Game modules cannot link `libc` — they will be compiled to QVM bytecode and run
inside the engine. So id ships their own subset in
[code/game/bg_lib.c](code/game/bg_lib.c): `memcpy`, `strcpy`, `qsort`, `sscanf`,
`malloc`, etc. There is no `FILE*` and no I/O.

> **Concept.** When you control the toolchain, you can prune the standard
> library to what is sound *for your purposes*. Embedded C, Rust `#![no_std]`,
> and Zig's `freestanding` target all do this. It is also what every safe-mod
> system has had to do since: untrusted code that links a "real" libc is one
> `system()` call away from owning the host.

---

## 6. The Virtual Machine and the Module ABI

> [STUDY_GUIDE.md](STUDY_GUIDE.md) §4 already gives the diagrams. This section
> is the textual deep-dive: opcode set, loader, syscall mechanics, JIT.

### 6.1 Why a VM at all?

In 1999, mods were a major reason FPS games sold. Quake 1 mods were QuakeC
bytecode. Quake 2 mods were native DLLs. Native DLLs are fast but: a bad mod
crashes the game, a malicious mod can do anything to the host, DLLs are
platform-specific, DLLs from the internet become a malware vector.

id's response in Q3 was a custom virtual machine — QVM — with about 50 opcodes;
bytecode files are magic-numbered and run in a sandboxed data segment. Because
the VM is hand-written, the engine can JIT-compile QVM bytecode to native x86
(or PowerPC) for near-native speed, while still falling back to interpretation
on platforms without a JIT.

You can think of QVM as a precursor to NaCl, asm.js, and eBPF.

### 6.2 The opcode set

[code/qcommon/vm_local.h](code/qcommon/vm_local.h) declares `opcode_t`:

- **Stack management**: `OP_ENTER`, `OP_LEAVE`, `OP_PUSH`, `OP_POP`, `OP_LOCAL`, `OP_ARG`
- **Constants and control flow**: `OP_CONST`, `OP_JUMP`, `OP_CALL`, `OP_BREAK`
- **Conditional branches**: `OP_EQ`, `OP_NE`, `OP_LTI`, `OP_GEU`, `OP_EQF`,
  `OP_GTF`, … — separate opcodes for signed int, unsigned int, and float
  comparison
- **Memory**: `OP_LOAD1/2/4`, `OP_STORE1/2/4`, `OP_BLOCK_COPY`, `OP_SEX8/16`
- **Integer arithmetic**: `OP_ADD`, `OP_SUB`, `OP_DIVI`, `OP_MULU`, `OP_BAND`,
  `OP_LSH`, …
- **Float arithmetic**: `OP_ADDF`, `OP_SUBF`, `OP_MULF`, `OP_DIVF`, `OP_NEGF`
- **Conversions**: `OP_CVIF` (int→float), `OP_CVFI` (float→int)

There is no `OP_INDIRECT_CALL` and no general `OP_RET`-from-arbitrary-stack:
control flow is structured. Every memory operation is masked by `dataMask` so
it cannot escape the VM's data segment — that mask is the entire sandbox.

### 6.3 Loading a module

[code/qcommon/vm.c:435](code/qcommon/vm.c#L435) — `VM_Create(module, systemCalls, type)`:

1. Read the `vm_*` cvar to decide between native DLL, interpreted bytecode, and JIT.
2. Load the `.qvm` file via the filesystem layer.
3. Validate the header magic (`VM_MAGIC == 0x12721444`, see `vmHeader_t` in
   [code/qcommon/qfiles.h](code/qcommon/qfiles.h)).
4. Allocate the data, BSS, and stack segments — all in a single power-of-two
   buffer so address masking is one AND instruction.
5. If JIT: invoke `VM_Compile()` from
   [code/qcommon/vm_x86.c](code/qcommon/vm_x86.c) to produce x86 machine code.
6. Store a function pointer to `systemCalls` (the engine's syscall dispatcher
   for this module) in `vm->systemCall`.

The first three lines of [code/qcommon/vm.c](code/qcommon/vm.c) tell the whole
story: the VM is a bridge between *our* world (engine) and *their* world
(module). Everything that crosses the bridge does so by an integer ID and a
stack of `int` arguments.

### 6.4 Syscalls — the ABI

A module never calls an engine function directly. Instead,
[code/game/g_syscalls.c](code/game/g_syscalls.c) defines wrapper functions like
`trap_Print(const char *text)` that ultimately invoke
[code/game/g_syscalls.asm](code/game/g_syscalls.asm), which contains a
hand-written `syscall` opcode that traps into the engine. The opcode passes a
syscall number from the `gameImport_t` enum in
[code/game/g_public.h](code/game/g_public.h):

```
G_PRINT, G_ERROR, G_MILLISECONDS,
G_CVAR_REGISTER, G_CVAR_SET, G_CVAR_VARIABLE_INTEGER_VALUE,
G_FS_FOPEN_FILE, G_FS_READ, G_FS_WRITE, G_FS_FCLOSE_FILE,
G_LOCATE_GAME_DATA, G_LINKENTITY, G_UNLINKENTITY,
G_TRACE, G_POINT_CONTENTS, G_IN_PVS, G_AREAS_CONNECTED,
G_SET_CONFIGSTRING, G_SEND_SERVER_COMMAND, G_GET_USERCMD,
... (~100 more, then BOTLIB_*, BOTLIB_AAS_*, BOTLIB_EA_*, BOTLIB_AI_* ranges) ...
```

The engine side ([code/server/sv_game.c](code/server/sv_game.c)) implements
`SV_GameSystemCalls(int *args)` as a giant switch on `args[0]`. The other
arguments are read off the stack: pointers from the module are translated by
adding the VM's data base. **VM addresses are offsets, not raw pointers.** This
is `VM_ArgPtr`, and it is the trust boundary.

The `cgame` module has its own enum `cgameImport_t` in
[code/cgame/cg_public.h](code/cgame/cg_public.h) and dispatcher in
[code/client/cl_cgame.c](code/client/cl_cgame.c). The `ui` module similarly.

> **Concept.** This is a stable, language-agnostic, transport-agnostic
> interface defined entirely by an integer ID and a flat argument array. Linux
> syscalls have the same shape. gRPC has the same shape with metadata. Wasm
> imports have the same shape with type signatures. eBPF helper functions have
> the same shape. Once you see the pattern, you see it everywhere.

### 6.5 The interpreter

[code/qcommon/vm_interpreted.c](code/qcommon/vm_interpreted.c) is the executable
specification of the VM. Read it before the JIT.

Implementation details to notice:

- The dispatch loop is a `goto *targets[opcode]` computed-goto table when
  supported, else a `switch`. Both forms eliminate the function-pointer-call
  overhead per instruction.
- The operand stack lives in the VM's own memory; the host stack is not used
  for VM computation.
- All memory access is masked: `data[address & dataMask]`. A wild pointer in
  the bytecode wraps around within the data segment instead of crashing the host.

### 6.6 The x86 JIT

[code/qcommon/vm_x86.c](code/qcommon/vm_x86.c) (1196 lines) compiles QVM
bytecode to x86 in two passes. `VM_Compile` is at line 399.

- **Pass 1**: walk the bytecode to learn the size of the emitted machine code
  and to record the byte offset where each bytecode instruction begins. This
  produces `instructionPointers[]`, used to translate `OP_JUMP` and `OP_CALL`
  targets.
- **Pass 2**: emit machine code into an executable buffer.

The emitter uses `Emit1`, `Emit2`, `Emit4`, and `EmitString` (see line 282)
that just `memcpy` opcode bytes into the buffer. Examples from line 307+:

```
EmitString( "89 07" );          // mov dword ptr [edi], eax
EmitString( "83 EF 04" );       // sub edi, 4
EmitString( "8B 1F" );          // mov ebx, dword ptr [edi]
```

There is no instruction-selection peephole optimizer beyond a few hand-written
cases. Most QVM ops map to one or two x86 instructions operating on EAX/EDX
with the operand stack kept in a fixed register-relative buffer addressed by
EDI.

> **Concept.** A JIT does not need to be HotSpot to be valuable. Even a
> one-pass macro-style JIT typically delivers 5–10× speedup over an interpreter,
> because the interpreter's *dispatch overhead* dominates everything else. If
> your interpreter spends 50 ns per opcode and your average opcode does 5 ns of
> real work, replacing dispatch with straight-line emission removes the 45 ns
> of overhead.

### 6.7 The native fallback

If `vm_game` is set to `0`, the engine loads `qagame_x86.dll` (or `.so` /
`.dylib`) and uses its `vmMain` and `dllSyscall` exports directly. The wrapper
macros in `g_syscalls.c` are written so the *same source* compiles in both
worlds: in the QVM build they emit a syscall opcode; in the native build they
call function pointers handed to the module at load time. Same trick as
NaCl/PNaCl a decade later.

---

## 7. Networking: Snapshots, Deltas, and Huffman

### 7.1 Design principles

- **The server is authoritative.** Clients never tell the server what happened;
  they tell the server what they *tried* (a `usercmd_t`: forward/right/up
  movement, view angles, buttons, weapon). The server runs the simulation and
  tells clients what resulted.
- **Communication is lossy and out-of-order.** UDP, no retransmission for game
  state. Reliability is reserved for *commands* (chat, configstrings).
- **State is serialized as a delta against a known baseline.** Each snapshot
  the server sends is a delta against a previously-acked snapshot. The client
  carries a 32-deep history (`PACKET_BACKUP`) of acknowledged snapshots and
  tells the server which one to delta against.
- **Bandwidth is precious.** The pipeline is bit-packed (not byte-packed) and
  Huffman-coded for further reduction.

### 7.2 The pieces

- `usercmd_t` — input from client. A few angles, three movement axes, button
  bitmask, weapon, server time.
- `playerState_t` — per-client authoritative state. The whole player: origin,
  velocity, animation, ammo, score, powerups…
- `entityState_t` — per-entity replicated state. All non-player entities (items,
  projectiles, doors, players visible from this client). Includes a
  `trajectory_t` so motion can be evaluated client-side at any time without
  storing every intermediate frame.

All three are defined in [code/game/q_shared.h](code/game/q_shared.h).

### 7.3 Bit-packed I/O

[code/qcommon/msg.c](code/qcommon/msg.c) is the serialization library:

- `MSG_WriteBits(msg, value, bits)` writes a signed or unsigned integer in
  exactly `bits` bits. Negative `bits` means signed, with sign extension on read.
- `MSG_WriteDelta(msg, oldV, newV, bits)` writes a single bit indicating
  "changed", followed by `bits` bits of new value if it changed.
- `MSG_WriteDeltaEntity(msg, from, to, force)` walks the `entityStateFields[]`
  table — a list of `{ field offset, bits, type }` triples — and emits each
  field that changed between `from` and `to`. **The field table is the schema
  of the wire protocol.**

> **Concept.** A schema-driven serializer keyed by a static table is faster
> than any reflection-based scheme and is provably size-bounded. Protobuf and
> Thrift use the same pattern. The table is defined once and shared by sender
> and receiver, which means no version negotiation is possible — every client
> must speak exactly the server's protocol version.

### 7.4 Snapshots, frame by frame

[code/server/sv_snapshot.c](code/server/sv_snapshot.c):

```
SV_SendClientSnapshot(client):
    SV_BuildClientSnapshot(client)
        ├── Determine which entities are visible to this client
        │   (PVS, area portals, snapshot callback in game.qvm)
        ├── Copy their entityState_t into svs.snapshotEntities[next slot]
        └── Record this client's clientSnapshot_t
    SV_WriteSnapshotToClient(client, msg)
        ├── Find a previously-acked snapshot to delta against (lastframe)
        ├── MSG_WriteDeltaPlayerstate(from->ps, to->ps)
        └── SV_EmitPacketEntities(from, to, msg)
              For each entity slot: same? skip. different? delta. removed? mark.
```

The server keeps a circular buffer of snapshots 32 deep. The client acks the
*snapshot number* it last received intact. The server uses that ack as the
"from" baseline for the next snapshot. If the network drops packets, the
from-baseline gets older, but as long as it's still inside the 32-deep window,
the snapshot remains decodable. If it falls out of the window, the server
sends a full ("uncompressed") snapshot.

### 7.5 Reliable side channel

Some events cannot tolerate loss: chat, scoreboard updates, configstring
changes. They travel through `Netchan_Transmit` as *reliable commands* —
appended to every outgoing packet until acknowledged. See
[code/qcommon/net_chan.c](code/qcommon/net_chan.c).

The `netchan_t` also handles **fragmentation**: messages larger than the MTU
are split into `FRAGMENT_SIZE`-byte chunks (1300 bytes) with an `is_fragment`
flag and a fragment offset, then reassembled on receive. The engine never has
to worry about path MTU because it never sends more than `FRAGMENT_SIZE` per
UDP packet.

### 7.6 Adaptive Huffman compression — close-up

[code/qcommon/huffman.c](code/qcommon/huffman.c) implements an *order-0
adaptive* Huffman coder. The whole file is 437 lines. The key functions:

| Line | Function | Role |
|---|---|---|
| 32 | `Huff_putBit` | Single-bit write into output buffer at a byte+bit offset |
| 42 | `Huff_getBit` | Single-bit read |
| 186 | `Huff_addRef` | Increment a symbol's frequency, rebalance the tree |
| 256 | `Huff_Receive` | Decode one symbol by walking the tree from root |
| 305 | `Huff_transmit` | Encode one symbol by emitting the path from root to leaf |
| 324 | `Huff_Decompress` | Decompress a `msg_t` in place from `offset` |
| 378 | `Huff_Compress` | Compress a `msg_t` in place from `offset` |
| 417 | `Huff_Init` | Build the initial tree from a baked-in frequency table |

Two important properties:

- **Adaptive.** Both endpoints start with the same tree (built in `Huff_Init`
  from a corpus of typical Quake messages — see the hard-coded weights at the
  top of the file). After every symbol they encode/decode, both sides call
  `Huff_addRef` to update the tree. As long as both sides update in the same
  order, they stay in sync without any explicit table being sent.
- **Bit-aligned.** The compressed stream is a bit-stream, not a byte stream.
  `Huff_putBit` and `Huff_getBit` track an `offset` measured in bits.

Compression typically yields another 30–50% reduction on top of the delta
encoding. The combination of (a) field-level delta encoding (b) bit-packed
field widths (c) adaptive Huffman is what made 28.8K dial-up multiplayer feel
responsive.

> **Concept.** Compression at the application layer is appropriate when (a)
> you control both endpoints and (b) you have domain knowledge that produces a
> better static probability model than a general-purpose compressor's runtime
> model. Q3's adaptive Huffman is initialized with hand-tuned weights for the
> exact byte distributions of Quake messages, and then refines them online.

### 7.7 Client-side prediction

[code/cgame/cg_predict.c](code/cgame/cg_predict.c) creates the illusion that
the local player responds *immediately* to input despite a 100ms+ round trip:

1. The client runs `Pmove()` on the local player every frame using the
   player's own `usercmd_t` (just generated by polling input). This is
   "prediction".
2. When a snapshot arrives, the client *replays* every `usercmd_t` not yet
   acknowledged, starting from the server's authoritative state. If the result
   agrees with what the client predicted, nothing visible happens. If it
   disagrees, the player snaps to the corrected position.

For *other* entities (other players, projectiles, items), the client
interpolates between the two most recent snapshots — see
`CG_TransitionEntity` and `BG_EvaluateTrajectory` in
[code/game/bg_misc.c](code/game/bg_misc.c).

[code/game/bg_pmove.c](code/game/bg_pmove.c) is compiled into both modules so
the prediction matches the authoritative simulation *bit-for-bit* (assuming
deterministic floating-point on identical hardware — see §12.3).

> **Concept.** Authoritative server + client prediction + entity interpolation
> is the trinity of every modern fast-paced multiplayer architecture: Source
> engine, Unreal replication, Overwatch's network model. They differ in
> details (rollback, lag compensation, tickrate) but the bones are Quake's.

### 7.8 The connection handshake — three phases on top of three datagrams

The high-level story is simple: there is a stateless ASCII pre-game that turns
into a sequenced binary protocol. The implementation is in three files acting in
concert.

**Phase 1 — out-of-band handshake.** A connectionless packet is recognised by a
sentinel sequence number of `-1` (i.e. four `0xFF` bytes). When
[Netchan_Process](code/qcommon/net_chan.c#L304) is *not* called — that is, when
the sequence number is `-1` — control falls to
[SV_ConnectionlessPacket](code/server/sv_client.c#L502) and
[CL_ConnectionlessPacket](code/client/cl_main.c#L1771). These functions
`Cmd_TokenizeString` the rest of the packet and dispatch on the first word:
`getchallenge`, `connect`, `getstatus`, `getinfo`, `rcon`, `disconnect`. Replies
go through [NET_OutOfBandPrint](code/qcommon/net_chan.c#L647) (text) or
[NET_OutOfBandData](code/qcommon/net_chan.c#L673) (Huffman-compressed payload —
the only place adaptive Huffman is used end-to-end).

The reason for the challenge round-trip is given verbatim in the source comment
at `SV_GetChallenge`: *"to prevent denial of service attacks that flood the
server with invalid connection IPs."* An attacker spoofing a source IP cannot
receive `challengeResponse` and so cannot craft a valid `connect` command, and
the server allocates only a tiny `challenge_t` slot until the round-trip
completes — not a full `client_t`. Modern equivalents: TCP SYN cookies,
QUIC's RETRY token.

**Phase 2 — gamestate transfer.** Once the netchan is open
([Netchan_Setup](code/qcommon/net_chan.c#L86), called from both
`SV_DirectConnect` and `CL_ConnectionlessPacket("connectResponse")`), the client
sends a `clc_clientCommand "new"`. The server replies with `svc_gamestate`: a
single sequenced message containing every configstring (`MAX_CONFIGSTRINGS = 1024`,
each up to ~1 KB) and an entityState baseline for every spawned non-player
entity. This message is typically 8–20 KB and triggers fragmentation (§7.9).
Once the client has acked it, both sides advance their `serverId` to a new
value — the gamestate has versioned the world, and any in-flight usercmds with
the old `serverId` are silently discarded.

**Phase 3 — steady state.** From now on every packet is sequenced, encoded,
Huffman-compressed, and carries either snapshots (server→client) or usercmds
(client→server) along with reliable commands.

| Client `cls.state` | Trigger | Server `cl->state` (mirror) |
|---|---|---|
| `CA_DISCONNECTED` | boot | `CS_FREE` |
| `CA_CONNECTING` | user typed `connect` | n/a |
| `CA_CHALLENGING` | got `challengeResponse` | `challenge_t` slot allocated |
| `CA_CONNECTED` | got `connectResponse` | `CS_CONNECTED` |
| `CA_LOADING` | got gamestate | `CS_PRIMED` |
| `CA_PRIMED` | media loaded | `CS_PRIMED` |
| `CA_ACTIVE` | first snapshot parsed | `CS_ACTIVE` |
| `CA_*` → `CA_DISCONNECTED` | timeout / kick | `CS_ZOMBIE` (silent for `sv_zombietime`) |

A client in `CS_ZOMBIE` is the Quake-style equivalent of a TCP `TIME_WAIT`
state: the server keeps the slot for a few seconds so any straggler packets
that contain old reliable commands don't get treated as a brand-new connection.

### 7.9 Fragmentation — the only stream protocol on top of UDP

`Netchan_Transmit` ([net_chan.c:246](code/qcommon/net_chan.c#L246)) makes the
fragmentation decision and is small enough to read entire:

- If `length < FRAGMENT_SIZE`, write a normal sequenced datagram and return.
- Otherwise, copy the data into `chan->unsentBuffer`, set `unsentFragments =
  qtrue`, and call [Netchan_TransmitNextFragment](code/qcommon/net_chan.c#L189)
  once. The remaining fragments go out *one per server frame* — the server's
  send loop (`SV_SendClientMessages` →
  [SV_Netchan_TransmitNextFragment](code/server/sv_net_chan.c#L134)) drains the
  pending fragments until done.

The fragment header is `4 (seq | FRAGMENT_BIT) + 2 (qport, client only) + 2
(start) + 2 (length)`. All fragments of one logical message share a sequence
number; the receiver reassembles into `chan->fragmentBuffer` and only delivers
the message when a fragment of length `< FRAGMENT_SIZE` arrives. There is one
quirk worth re-reading the source for: if the message is *exactly* a multiple of
`FRAGMENT_SIZE`, the sender must transmit a final 0-byte fragment so the
receiver knows the message is complete (see [net_chan.c:231](code/qcommon/net_chan.c#L231)).

**Loss handling is brutally simple.** If fragment N is lost, the receiver
discards every later fragment of that sequence and waits for the *whole*
message to be retransmitted (the next time the server sees the message
in-flight, the unacked sequence tells it to resend). There is no NACK, no
selective retransmission. This is acceptable because fragmentation in Q3 is
overwhelmingly a once-per-map event (gamestate). Steady-state snapshots almost
never fragment — they're tuned to fit in a single datagram.

> **Concept.** Q3 builds a tiny "stream protocol" on top of UDP for the one use
> case where it's needed (large gamestate), and pays no protocol-state cost for
> the 99% of packets that don't need it. Compare with QUIC's stream framing,
> which pays per-stream metadata cost on every datagram even when there's only
> one stream of one packet.

### 7.10 The reliable command channel — sliding window of text

Q3 needs to deliver short text commands (`chat hi`, `cs 12 "newconfig"`,
`disconnect`) with TCP-style guarantees over a UDP-style transport. The trick
is to layer the channel inside the same datagram stream and skip the
machinery of a separate retransmit timer:

```
Sender side state                 Receiver side state
  reliableSequence  (next slot)     lastClientCommand (highest seq executed)
  reliableAcknowledge (last acked)  reliableCommands[seq & 63]   (cache for re-exec dedup)
  reliableCommands[seq & 63]
```

On every packet, the sender writes **every command between `reliableAcknowledge
+ 1` and `reliableSequence`** ([SV_UpdateServerCommandsToClient](code/server/sv_snapshot.c#L208) for s→c,
[CL_WritePacket](code/client/cl_input.c#L722) for c→s). On every packet, the
receiver writes back the highest command sequence it has executed. Idempotency
is enforced by `if (seq <= lastClientCommand) continue;` —
[SV_ClientCommand](code/server/sv_client.c#L1258) explicitly handles
re-executions of an already-acked command.

A clever consequence: **packet loss costs you bytes, not state**. If a packet
is dropped, the next packet just naturally repeats the same commands. There is
no NACK, no fast retransmit, no head-of-line blocking. The bandwidth overhead
is `O(unacked-commands × packet-rate)` — fine for chat (tens of bytes), and
flagged as "lost reliable commands" when the unacked window exceeds
`MAX_RELIABLE_COMMANDS = 64`, at which point the receiver drops the connection.

Two important consequences cascade out of this design:

- **The reliable command sequence is the integrity baseline for the XOR
  obfuscation in §7.11.** Both sides agree on the most recently acked command
  text, and that text feeds the keystream. This binds the obfuscation to game
  state — an attacker proxy cannot decrypt later packets without observing all
  earlier ones in order.
- **Rate limiting is per-command, not per-byte.**
  [SV_ClientCommand](code/server/sv_client.c#L1289) sets `cl->nextReliableTime
  = svs.time + 1000` on every accepted command and ignores commands that arrive
  earlier — one command per second per client is the floor, defeating chat
  spam without dropping the connection.

### 7.11 XOR obfuscation — the released-source surprise

Both [client](code/client/cl_net_chan.c#L38) and
[server](code/server/sv_net_chan.c#L36) wrap the post-netchan payload in an
XOR stream cipher whose keystream is derived from per-connection secrets:

```
key_byte(i) = challenge                              // 32-bit nonce from handshake
            ^ outgoingSequence                       // (server side) or
              (serverId ⊕ messageAcknowledge)        // (client side)
            ^ (last_acked_reliable_command)[i % len] // shifted by (i & 1)
```

The first 12 bytes (server side: 4 bytes) are skipped so the receiver can
read `serverId`, `messageAcknowledge`, and `reliableAcknowledge` *before*
deriving the key for the rest. High-bit and `%` characters in the command text
are normalized to `'.'` before being XOR'd in (defending against specific
chosen-plaintext patterns).

This is **not cryptography** — it's an obfuscation barrier targeted at packet
sniffers and bandwidth proxies. Anyone who can replay the connection from
`getchallenge` onwards and track every reliable ack can reconstruct the
keystream. But that's a much higher bar than the typical "proxy that
intercepts UDP and edits player coordinates", which is what id wanted to break.

The released-source layer is a redesign of the older
`Netchan_ScramblePacket` / `UnScramblePacket` permutation in
[net_chan.c:97-180](code/qcommon/net_chan.c#L97-L180), which is `#if 0`-ed out.
The original used a deterministic byte permutation seeded by the packet's first
4 bytes plus its length — security-by-shuffling, broken in minutes by anyone
with a hex editor. The replacement keys the cipher to the dynamic per-packet
acknowledgement state, so the keystream desyncs with even one-frame
divergence between sniffer and game.

> **Concept.** Game-protocol obfuscation has a different cost/benefit curve
> than transport encryption. The threat model is "casual aimbots and
> bandwidth-counter proxies", not "nation-state in the middle". A perfectly
> ordinary stream cipher keyed to slow-moving game state is enough for the
> threat model and avoids the ~µsec per-packet cost of real crypto on a 1999
> CPU.

### 7.12 Static-tree Huffman — and why §7.6 above slightly oversimplifies

The code above ([huffman.c](code/qcommon/huffman.c)) implements an *adaptive*
Huffman algorithm — both `Huff_Compress`/`Decompress` and the per-symbol
`Huff_offsetTransmit`/`Receive`. But the *connected packet path* never calls
`Huff_addRef` between symbols. Tracing the call tree:

```
MSG_WriteBits (msg.c:107)
  └── Huff_offsetTransmit(&msgHuff.compressor, byte, ...)    // tree never updates
MSG_ReadBits  (msg.c:184)
  └── Huff_offsetReceive(msgHuff.decompressor.tree, ...)     // tree never updates
```

`msgHuff` is built once at boot by [MSG_initHuffman](code/qcommon/msg.c#L1709):

```c
Huff_Init(&msgHuff);
for (i = 0; i < 256; i++) {
    for (j = 0; j < msg_hData[i]; j++) {
        Huff_addRef(&msgHuff.compressor,   (byte)i);
        Huff_addRef(&msgHuff.decompressor, (byte)i);
    }
}
```

`msg_hData[]` ([msg.c:1450](code/qcommon/msg.c#L1450)) is a hand-tuned 256-entry
frequency table. Roughly: byte 0 has weight ~250000 (sentinels and zero
deltas dominate Q3 packets), bytes representing common opcode IDs cluster
in the 5000-30000 range, and obscure bytes get weight 1. The static tree
built from this table is *both endpoints' shared codebook* and is never
rebalanced once the engine starts.

So the "adaptive Huffman" claim in marketing material and in jfedor's writeup
is technically true *only of the initial `connect` userinfo-bearing OOB
packet* (which goes through `Huff_Compress` / `Huff_Decompress` and walks an
adaptive tree from a fresh NYT). Every subsequent connected packet uses a
fixed tree.

Why a fixed tree wins:

- **No state to lose on packet drop.** Packet loss does not desync the
  decoder; if it did, every dropped UDP packet would corrupt every following
  one.
- **Decode speed.** Symbol decode is a tree walk; with a static tree,
  branch-prediction warms up; with an adaptive tree, every symbol reshuffles
  the upper levels.
- **Determinism.** The protocol version is a property of the binary; both
  sides have to ship the same `msg_hData[]` to interoperate. This is a
  feature: protocol-version skew becomes a build issue, not a runtime
  negotiation.

The cost is suboptimal compression on byte distributions that drift from id's
reference corpus. The corpus was sampled in 1998-99 from in-house playtests —
modern mods that pack bizarre data into entityState fields will compress
worse than the engine assumes, but rarely badly enough to matter.

### 7.13 Bandwidth control: rate, snaps, and the snapshot scheduler

Three knobs interact to set the actual snapshot rate per client:

| cvar | Side | Default | Role |
|---|---|---|---|
| `rate` | client (sent in userinfo) | `3000` bytes/sec | the client tells the server how much it can absorb |
| `snaps` | client (userinfo) | `20` snapshots/sec | upper bound on snapshot rate |
| `sv_maxRate` | server | `0` (off) | server overrides per-client `rate` if non-zero |
| `sv_fps` | server | `20` | server simulation rate; snapshots can't exceed this |

The scheduler is in [SV_SendClientMessages](code/server/sv_snapshot.c#L655):

```c
for each client:
    if (svs.time < c->nextSnapshotTime) continue;     // snaps cap
    if (c->netchan.unsentFragments) {
        send next fragment of in-flight gamestate;
        nextSnapshotTime = svs.time + RateMsec(c, remaining_fragment_size);
        continue;
    }
    SV_SendClientSnapshot(c);
```

`SV_SendClientSnapshot` calls [SV_RateMsec](code/server/sv_snapshot.c#L527),
which computes how long that snapshot's bytes "should take" to transit the
client's `rate` budget:

```c
rateMsec = (messageSize + 48) * 1000 / rate;  // 48 = ~IP+UDP+netchan overhead
```

If `rateMsec` exceeds `client->snapshotMsec` (`= 1000 / snaps`), the next
snapshot is delayed and `SNAPFLAG_RATE_DELAYED` is set on the *next* packet so
the client's lagometer can paint a yellow bar. Otherwise the snapshot rate is
held at `snapshotMsec`.

There are two escape hatches:

- **Loopback / LAN clients** (`sv_lanForceRate`) get a snapshot per server
  frame, no `rate` accounting. Local-server play has no network cost.
- **Bots** (`SVF_BOT`) have `SV_BuildClientSnapshot` called on them so they
  see the world, but [SV_SendClientSnapshot](code/server/sv_snapshot.c#L619)
  early-returns before encoding — bots query their own snapshot in-process,
  no UDP round-trip.

> **Concept.** The snapshot scheduler is a *credit-based* rate limiter, not a
> token bucket: each snapshot consumes credit proportional to its compressed
> size, and credit refills at `rate` bytes/sec. This is the simplest possible
> way to convert "bytes/sec budget" into "send delay" and avoids any per-tick
> accounting state.

### 7.14 Client-side prediction — the replay loop in detail

§7.7 above sketches the high-level idea. Here is the actual mechanism, with
file references.

The client maintains two playerstates:

- `cl.snap.ps` — the last authoritative state from the server.
- `cg.predictedPlayerState` — the client's local extrapolation. This is what
  the renderer reads.

Every frame, [CG_PredictPlayerState](code/cgame/cg_predict.c) does:

```
if (this is the first snapshot since last predict):
    start = cl.snap.ps                            // server's word is law
    first_unacked_cmd = cl.snap.ps.commandTime + 1
else:
    start = cg.predictedPlayerState                // continue from where we were

for each usercmd from first_unacked_cmd to cl.cmdNumber:
    Pmove(start, cmd)                              // run shared movement code
                                                   // (same Pmove the server runs)

cg.predictedPlayerState = start
```

The crucial property: `Pmove` ([bg_pmove.c](code/game/bg_pmove.c)) is
**bit-identical** between the client engine and `game.qvm`, because both are
compiled from the same source file. As long as floating-point semantics match
on both sides (no `--ffast-math`, no x87 80-bit intermediates differing from
SSE 32-bit, no different `-O` levels reordering FP), the prediction agrees
with the server. When it doesn't agree — collision with a moving door, damage
from another player — the client snaps; the player perceives a "lag spike" or
"warp", and that is what an authoritative architecture costs.

For *other* entities (other players, projectiles), the client doesn't predict
— it interpolates. Each `entityState_t` carries a `trajectory_t`
(`{trType, trTime, trBase, trDelta, trDuration}`) which is enough for
[BG_EvaluateTrajectory](code/game/bg_misc.c) to compute that entity's position
at any time `t` between two snapshots. The client pretends the world is
`snapshotInterval` ms in the past — typically 50–100 ms — so it always has
two snapshots to interpolate between. On packet loss, it briefly extrapolates
forward (the lagometer turns yellow), and snaps back when the next snapshot
arrives.

A subtle bonus: the same `trajectory_t` also describes ballistic projectiles,
falling debris, swinging doors, and conveyor belts. **Q3 does not transmit
positions for moving objects** — it transmits trajectories, and the client
evaluates them locally. A grenade in flight gets one entityState update at
launch and another at detonation; everything in between is computed by both
sides from the same trajectory.

### 7.15 The lagometer — instrumentation as feature

Q3's lagometer ([cg_draw.c:1580](code/cgame/cg_draw.c#L1580)) is two stacked
graphs at the bottom-right of the HUD, each backed by a 128-sample circular
buffer:

- **Top graph (`frameSamples[]`).** One sample per render frame, recording
  `cg.time − latestSnapshotTime`. Negative = interpolating between two known
  snapshots (good, painted green). Positive = extrapolating into the future
  past the last snapshot (bad, painted yellow). The graph turning yellow is
  the visible signature of "snapshot was late or lost; the world you see is
  the client's guess."
- **Bottom graph (`snapshotSamples[]`).** One sample per snapshot received,
  recording `snap->ping`. Gaps = dropped snapshots. A yellow tint on the
  sample = `SNAPFLAG_RATE_DELAYED`, meaning the *server* throttled it for
  `rate` budget reasons (i.e. you, the player, asked for too low a `rate`).

Two pieces of design wisdom worth flagging:

1. **The lagometer is implemented in `cgame.qvm`, not the engine.** It runs as
   bytecode inside the client VM, has no special privileges, and could be
   reimplemented or replaced by a mod. The engine merely exposes `getCmd`,
   `latestSnapshotTime`, and a bitmap shader.
2. **The graph is not just a debugging aid — it's a UX contract.** When the
   player sees yellow, they know the warp wasn't their fault. That
   psychological framing reduces support tickets and matters in a competitive
   game where every disagreement about network conditions becomes a forum
   thread.

> **Concept.** Make your protocol's flaky-network behaviour *visible* to the
> user. Modern equivalents: Discord's "weak connection" indicator, Zoom's
> network-strength dot, browser dev tools' DevTools ⟶ Network tab. Q3 had this
> in 1999 because the protocol was tuned for 56k modems where flakiness was
> normal.

### 7.16 Lag compensation: the Unlagged mod

Vanilla Q3 has the netcode flaw most discussed by competitive players: **you
shoot in the predicted present, but the targets you aim at are rendered in
the interpolated past**. The visual gap between "where the railgun crosshair
points" and "where the server thinks the opponent is" equals
`ping + cl_interp_delay`, typically 100–200 ms on the open internet. Hitscan
weapons (railgun, machinegun, shotgun) become exercises in lead-prediction.

[Neil Toronto's Unlagged mod](https://www.ra.is/unlagged/) (2003) introduced
the same fix Valve later shipped in Half-Life 1 and Counter-Strike: **server
side time-shifted hit detection**, also known as lag compensation or backward
reconciliation.

**The history buffer.** The server-side game module hooks the end of every
`G_RunFrame` and snapshots each `playerState_t.origin` into a per-client
circular buffer indexed by server time. Per the unlagged docs:

> "The server needs to remember what states every player has been in for
> every snapshot, back a few hundred milliseconds. […] the stored value is
> the *final*, *snapped* origin — the actual origin sent to every client."

The detail about *snapped* origins matters: client-side prediction operates
on the rounded values that came over the wire (`PS_ORIGIN_BITS` truncates
the float), so the server must rewind to the same rounded values for the
trace to agree with what the shooter saw. A naive implementation that
rewound to the floating-point origin would still produce subtle disagreements
in edge-of-bbox cases.

**The trace-time rewind.** When a hitscan weapon fires, the server runs a
three-step pattern around the existing `trap_Trace` call. In pseudocode
(actual function names from unlagged: `G_TimeShiftAllClients`,
`G_UnTimeShiftAllClients`):

```c
void G_FireHitscan(client_t *attacker, vec3_t origin, vec3_t dir) {
    int target_time = level.time
                    - attacker->ping
                    - attacker->cgame_interp_delay
                    - 50;          // Q3's snapshot grain

    G_TimeShiftAllClients(target_time, attacker);
        // for each other client c:
        //     save c->r.currentOrigin
        //     c->r.currentOrigin = c->history[target_time]
        //     trap_LinkEntity(c)        // re-link bbox

    trap_Trace(&tr, origin, mins, maxs, end, attacker - g_entities, MASK_SHOT);

    G_UnTimeShiftAllClients(attacker);
        // for each shifted client:
        //     restore saved origin
        //     trap_LinkEntity(c)

    if (tr.entityNum < MAX_CLIENTS) {
        // hit registered
    }
}
```

Three details that make or break the implementation:

1. **The shooter is not rewound.** Their movement reaches the server before
   their fire command (because cmds are sent with the *cmd's* serverTime,
   which includes the predicted `serverDelta`). The shooter's server-side
   position therefore already matches what they predicted client-side.
2. **The window must be tight.** "It is important that this
   backward-reconciled state last as short a time as possible." Anything
   else that runs while the world is rewound — a triggered door, a detpack —
   would see the wrong state. The unlagged code is structured so the
   rewind/restore is a tight bracket around exactly the trace call.
3. **Projectiles need different treatment.** A hitscan trace is instant;
   a rocket flies for 1.5 seconds. Unlagged compensates only the *spawn
   position* of projectiles (so your rocket leaves the muzzle where you
   aimed it) — the rocket then flies under normal physics from then on. A
   target you "saw" hit at fire-time may dodge the rocket; that's correct.

**The `cg_smoothClients` companion fix.** With lag compensation in place,
prediction of *other* players actively damages accuracy: the local
prediction will move other players to where the local copy of `Pmove` thinks
they should be, which differs subtly from the server's view. Unlagged adds
`cg_smoothClients 1`, which forces the client to *only* predict its own
player and *only* interpolate everyone else. Now the rendered positions of
opponents exactly match the snapshot stream, which exactly matches the
server's history buffer, which is what the trace runs against. The whole
chain is internally consistent.

| Cost | Vanilla Q3 | Unlagged |
|---|---|---|
| Server CPU per shot | 1 trap_Trace | 1 trap_Trace + 2 × N origin overwrites + 2 × N relinks |
| Server memory per client | 0 | ~120 bytes (history buffer) |
| Shooter UX | "I have to lead my targets" | "what I see is what I hit" |
| Victim UX | "low-ping wins, fair" | "I died after hiding behind cover" |
| Cross-client fairness | ping-fair | shooter-fair |
| First shipped | n/a | 2003 (mod), backported to OSP/CPMA shortly after |

The "killed around the corner" complaint that originates here is a real
artifact: the high-ping shooter saw the target standing in the open, the
low-ping target saw themselves duck behind cover, and the server (which
only knows shooter-saw-state) registered the hit. The trade is deliberate —
you give up perfect shared reality to give up perfect-aim-as-a-function-of-
ping. **Modern competitive shooters (CS2, Valorant, Overwatch 2, Apex)
universally use this same scheme.** Q3 was the engine where the dispute was
first articulated.

> **Concept.** "Authoritative server" is not one thing. There is a spectrum
> from "server simulates everything, clients are dumb terminals" (Quake 1)
> through "server is authoritative but rewinds for hit detection" (Q3 with
> Unlagged, modern competitive games) through "server doesn't even rewind,
> just trusts client-predicted state and validates" (some MMOs). Where you
> sit on this spectrum is a tradeoff between cheating-resistance, fairness
> across pings, and how much "what you see is what you hit" matters to the
> genre.

### 7.17 Forward to Doom 3 — id's own critique of Q3's netcode

In 2006 J.M.P. van Waveren of id Software published
[The DOOM III Network Architecture](https://mrelusive.com/publications/papers/The-DOOM-III-Network-Architecture.pdf),
a 27-page paper that explicitly enumerates what id thought was wrong with
Q3's netcode by 2004 and what they redesigned for Doom 3. It is the most
authoritative critique of Q3 in print, and reading it changes how you
understand the trade-offs in the Q3 source.

The headline reframe: **Q3 puts the client behind the server, Doom 3 puts
the client ahead of the server.**

```
Q3 timeline:
                                              server (svs.time) ──────────►
                            other entities visible at: svs.time − interp_delay
                                       player physics at: svs.time + RTT/2 (predicted)
                                                                ▲
                                          "shoots in present, sees past"

Doom 3 timeline:
   server time ──────────►
                     ▲ snapshots arrive at this offset
   client time ─────────────────►   (runs ahead by ping + jitter margin)
                                ▲ player input here
                                  → travels to server, arrives just in time
```

The architectural changes that follow from this reframe are sweeping. The
paper makes seven concrete critiques of Q3:

**1. Separate game/cgame modules with strong coupling.** In Q3, server
logic lives in `game.qvm` and client visualization + prediction lives in
`cgame.qvm`. They share `bg_*.c` source but compile into separate VMs with
separate state. The paper notes this makes single-player development
awkward: "developers often released two executables — one for multiplayer
based on the original sources, and another for single-player that bypasses
the separation". Doom 3 collapses both into one `idGame` module that runs
on both server and client, identically.

**2. Fixed entityState_t.** Q3's `entityState_t` ([msg.c:786-839](code/qcommon/msg.c#L786-L839))
is a 50-field struct, every field of which is sent for every entity that
needs network presence. In practice, devs end up reusing fields for
different purposes on different entity types (`generic1`, `time2`, etc).
Doom 3 replaces this with a per-entity `idBitMsg` — each entity writes
its own state to the wire in whatever schema fits. The cost of the
flexibility is per-entity logic; the gain is no field reuse and no schema
churn.

**3. Reliable commands as ASCII text.** Q3's reliable channel encodes every
command as a ASCII string parsed via `Cmd_TokenizeString`. This is fine for
chat but absurd for synchronizing small chunks of game state ("player N's
skin changed to red"). Doom 3's reliable channel carries arbitrary binary
payloads and piggybacks on every unreliable message until acked, with no
retransmit timer (the client send rate is 3-4× the server snapshot rate, so
"if the next four packets don't ack, something's wrong" is a sufficient
heuristic).

**4. Delta only against the previous snapshot.** Q3's `clientSnapshot_t`
ring is 32 deep and the server delta-encodes the new snapshot against the
last one the client acked. **Entities that leave and re-enter the PVS
have to be sent in full each time** because the receiver has no memory of
their prior state. In an open-area game this is fine; in a corridor-heavy
one with lots of corner-cutting (Doom 3), it's a serious bandwidth cost.

Doom 3's fix is a *common base state* maintained on both sides. When the
server applies a snapshot to its common base, it sends a reliable message
telling the client to do the same. Now both sides have an always-current
"reference state" for *every* entity, not just those currently in the PVS.
An entity entering the PVS now delta-encodes against its last-known state
in the common base — usually a small delta even if it's been gone for
seconds.

**5. PVS membership is implicit.** In Q3, an entity is "in the PVS" iff it
appears in the snapshot's entity list. There is no explicit "this entity
is no longer visible" message; the client has to infer it from the
delta-remove flag. Doom 3 adds an explicit 4096-bit PVS membership string,
delta-compressed in 32-bit chunks (so a typical scene where 3 entities
enter or leave costs 3 × 4 bytes = 12 bytes of membership signal, not 512).

**6. Lag compensation is "inherently flawed".** The paper's most quotable
section, on the kind of fix Unlagged represents:

> "Lag compensation is also inherently flawed because it can cause events
> to happen that do not reflect the shared reality. […] The player with the
> low ping can be shot which from the perspective of that player should not
> be possible."

Doom 3 instead leverages **determinism**. Both sides run the same simulation
code at a fixed 60 Hz. The server sends each snapshot the most-recent
usercmd from every player in the PVS. The client uses those usercmds to
*predict ahead* every entity in its PVS, not just the local player. Because
the same physics + input produce the same output, the client's predicted
positions of opponents are usually accurate, and the player can shoot in
the present and see in the present.

**7. Q3 client extrapolation is unsafe.** When snapshots are late, Q3's
`BG_EvaluateTrajectory` linearly extrapolates entity positions, ignoring
collision. The paper notes this lets entities visually pass through walls
during lag spikes. Doom 3's prediction runs the *full physics simulation*
forward — collisions, animations, AI — so extrapolated positions never
violate constraints.

The numerical scorecard from the paper:

| Concern | Q3 | Doom 3 |
|---|---|---|
| Game tick rate | 20 Hz | 60 Hz |
| Snapshot rate | 10–30 Hz (default 20) | 10–20 Hz |
| Client cmd rate | 30 Hz default | ~60 Hz (ahead-of-server) |
| Entities in scene | hundreds | thousands (more state to delta) |
| Snapshot baseline | last acked snapshot | shared common base + reliable ack |
| Delta survives PVS leave | no | yes |
| PVS membership | implicit (per-entity flag) | explicit 4096-bit string, RLE'd |
| Reliable channel | ASCII via cmd parser | binary piggyback |
| Compression | static-tree Huffman | bit-pack + delta + 3-bit-zero-RLE (max 4:1) |
| Single-module game logic | no | yes |
| Lag compensation | bolt-on (Unlagged) | none (predicted by determinism) |

> **Concept.** Game netcode evolves by a series of swings between
> "client-as-dumb-terminal" and "client-as-eager-predictor" extremes. Q1
> was almost-pure dumb terminal; Q3 went heavy on prediction for the local
> player only; Doom 3 pushed prediction to all entities; Source engine
> walked back to Q3's split with industrial-strength lag comp; modern
> competitive shooters (Valorant, CS2, Overwatch 2) re-converged on the
> Q3+Unlagged design because their networks are now fast enough for
> shooter-fair to feel ok. Each generation re-derives the trade-off as
> bandwidth, latency, and CPU budgets shift. Reading the Doom 3 paper
> alongside the Q3 source is the best way to internalize that the design
> space has more than one local optimum.

### 7.18 Reading the actual numbers — the Sanglard observation

Fabien Sanglard, in his [Q3 Source Code Review](https://fabiensanglard.net/quake3/network.php),
makes one quantitative observation worth flagging. Each `clientSnapshot_t`
carries the playerstate plus an entity index list, and the
`svs.snapshotEntities[]` ring sized to `MAX_SNAPSHOT_ENTITIES * sv_fps *
sv_maxclients * PACKET_BACKUP` slots. For a 4-player server at 20 Hz with
1024 entities visible per frame:

```
1024 entities × ~60 bytes (entityState_t) × 32 (PACKET_BACKUP) × 4 clients
≈ 8 MB of snapshot history
```

That's ~8 MB of RAM per server — meaningful in 1999, trivial today. But
the design choice it forces — **the server must keep every entity it has
*recently* sent to every client, just to delta against** — is the reason
Q3 cannot scale to 64+ player servers without significant memory pressure.
Doom 3's common-base approach has constant memory cost per entity rather
than per-(entity × client × snapshot), which is the second motivation
(after PVS-leave handling) for the redesign.

> **Concept.** Memory layout decisions in network protocols are often
> hidden in the data shape, not the wire protocol. Q3's
> `(client × PACKET_BACKUP × entities)` ring is invisible from the wire
> but fixes the upper bound on player count more tightly than any
> bandwidth calculation. When you're designing a replication protocol,
> sketch the per-server state size before sketching the packet format.

---

## 8. The Renderer in Brief

> [STUDY_GUIDE.md](STUDY_GUIDE.md) §10 has full Mermaid diagrams for the
> renderer pipeline, the shader-class hierarchy, and the per-frame call tree.
> This section emphasizes a few angles the visual guide compresses.

### 8.1 Frontend / backend split, again

The frontend ([code/renderer/tr_main.c](code/renderer/tr_main.c)) and backend
([code/renderer/tr_backend.c](code/renderer/tr_backend.c)) communicate by a
queue of commands. There are *two* `backEndData[SMP_FRAMES]` buffers. On a
single-CPU build, frontend fills buffer N and then backend drains it. On a
dual-CPU build (`r_smp 1`), frontend fills buffer N+1 *while* backend drains
buffer N.

> **Concept.** Pipelining a renderer between CPU-side scene assembly and
> GPU-side drawing is now standard practice (Vulkan command buffers, DX12
> bundles, Metal command encoders). Q3 was doing it for dual-Pentium III
> workstations.

### 8.2 Shaders that are not shaders

There were no programmable shaders on consumer GPUs in 1999. Q3's "shaders"
are *text scripts* that describe a multi-pass rendering recipe for a surface:
which textures to sample, in what order, with what blend modes, with what UV
transformations. [code/renderer/tr_shader.c](code/renderer/tr_shader.c) (3013
lines) parses these scripts.

A shader has multiple `shaderStage_t`s; each stage describes:

- the texture map(s) to bind,
- a `tcGen` (lightmap, environment-mapped, …),
- one or more `tcMod`s (rotate, scroll, scale, turbulence, …),
- an `rgbGen` and `alphaGen` (constant, vertex color, waveform, identity, …),
- a blend function and depth options.

Each stage corresponds to one OpenGL pass over the surface's geometry.
Multi-texturing (`GL_ARB_multitexture`) collapses two stages into one pass when
possible.

The list of supported stage keywords *is* the language definition. Reading the
parser is the fastest way to learn the shader DSL. The well-known *Q3 Shader
Manual* is essentially extracted from this code.

### 8.3 The OpenGL abstraction

[code/renderer/qgl.h](code/renderer/qgl.h) declares `qgl*` function pointers
for every GL entry point. The platform layer fills them at startup using
`wglGetProcAddress` or `dlsym`. The renderer never directly calls `glBegin`;
it calls `qglBegin`. This makes it possible to swap in a different GL
implementation (a software emulator, or a wrapper that logs every call for
debugging).

---

## 9. Collision Detection

[code/qcommon/cm_trace.c](code/qcommon/cm_trace.c) implements `CM_BoxTrace`
and `CM_TransformedBoxTrace`, the engine's swept-AABB-against-world primitive.
This function is called every time anything moves: bullets, players, items,
doors, particles.

The algorithm:

1. Recursively descend the BSP tree, splitting the sweep at each plane the
   ray crosses.
2. At each leaf, test the swept AABB against every brush in that leaf.
3. A brush is a convex polyhedron defined by a set of planes; the
   swept-AABB-vs-brush test is the classic *slab* algorithm extended for AABBs.
4. Curved surfaces are tessellated into triangle soup at level load
   ([code/qcommon/cm_patch.c](code/qcommon/cm_patch.c)) and tested separately.

The function is performance-critical (called thousands of times per frame).
Notice the heavy use of bounding-box rejection, plane-distance precomputation,
and the `qboolean isPoint` fast path for ray (not box) traces.

> **Concept.** BSP-broadphase + brush-narrowphase is one of three historically
> dominant collision architectures (the others: grid hashing, BVH). The right
> choice depends on the geometry: BSP is unbeatable for static world geometry
> that does not change after load.

---

## 10. Sound — the part the visual guide skips

The audio subsystem lives in [code/client/snd_*.c](code/client/) and is
deliberately small. Total: ~2700 lines in
[code/client/snd_dma.c](code/client/snd_dma.c) (1636),
[code/client/snd_mix.c](code/client/snd_mix.c) (681), plus support for ADPCM
and wavelet decompression.

### 10.1 Architecture

Two clocks, both measured in *sample pairs* (a "frame" of stereo audio):

- `s_paintedtime` — how many sample pairs have been written to the DMA buffer
  by the mixer.
- `s_soundtime` — how many sample pairs the audio hardware claims to have
  played (read from the OS layer, e.g. `SNDDMA_GetDMAPos()`).

The mixer's job is to keep `s_paintedtime` ahead of `s_soundtime` by enough
samples that the buffer never starves, but not so far ahead that newly-issued
sounds are noticeably delayed. Each frame, the mixer:

1. Reads `s_soundtime` from the OS.
2. If `s_paintedtime - s_soundtime` is too small, paints (mixes) more samples
   into the DMA ring buffer up to a target latency.
3. Returns; the OS continues to DMA whatever has been painted.

This is a classic *pull-when-asked* audio model, and the same one used by
DirectSound, OSS, and every modern low-level audio API.

### 10.2 The mixer

[code/client/snd_mix.c](code/client/snd_mix.c) does the painting. For each
active channel:

- Resample the source from its native sample rate to the output rate (linear
  interpolation, no anti-aliasing — the cost-quality knob is turned firmly
  toward cost).
- Apply per-channel volume (computed from listener distance, orientation, and
  occlusion in `S_Spatialize`).
- Mix into the output buffer with saturating add.

Looping sounds (engines, fans, wind) are handled separately in
`S_AddLoopSounds` so they don't have to rebuild a new channel every frame.

### 10.3 The "channel" abstraction

A `channel_t` is a *playing* sound: a pointer to an `sfx_t` (the loaded
waveform), a current sample position, gain, and listener-relative position.
The engine picks a free channel from a fixed pool (`MAX_CHANNELS`); when the
pool is full, the *least audible* channel is evicted. This is why dozens of
shotgun pellets all firing at once never overflow audio resources — the
quietest tail of the oldest pellet is silently dropped.

> **Concept.** A bounded resource pool with priority-based eviction is the
> single most useful pattern in real-time media systems. You see it in audio
> mixers, video decoders, particle systems, and (with a different name)
> connection pools and thread pools.

### 10.4 ADPCM and wavelet

`snd_adpcm.c` decodes 4-bit ADPCM samples — a 4:1 compression with audible
but acceptable quality, used to halve the on-disk size of dialogue and
ambient loops. `snd_wavelet.c` implements a wavelet-based codec for music
streams. Both are quite small; reading them is a good warm-up for media
decoders generally.

---

## 11. Bot AI

The bot system is its own field of study — the most sophisticated piece of
code in the repository, and one of the most influential bot AIs in game
history. 32,000 lines split between [code/botlib/](code/botlib/) (engine-side)
and the `ai_*` files in [code/game/](code/game/) (module-side).

> [STUDY_GUIDE.md](STUDY_GUIDE.md) §11 has the architecture diagrams and the
> bot state-machine. The textual summary of the layers:

- **AAS (Area Awareness System)** — [code/botlib/be_aas_*.c](code/botlib/) — a
  precomputed navigation graph built from the BSP by the standalone tool
  [code/bspc/](code/bspc/). Every walkable region of the level is one *area*;
  areas are connected by *reachabilities* (walk, jump, rocket-jump, teleport,
  etc.). Travel time between any two areas is precomputed and cached.
- **Movement** ([code/botlib/be_ai_move.c](code/botlib/be_ai_move.c)) —
  translates "go to area X" into per-frame movement commands by predicting
  player physics forward.
- **Goals** ([code/botlib/be_ai_goal.c](code/botlib/be_ai_goal.c)) — assigns a
  *weight* to every item, choke point, and player and picks the highest.
- **Weapon selection**
  ([code/botlib/be_ai_weap.c](code/botlib/be_ai_weap.c)) — fuzzy-logic
  table-driven; picks the best weapon for a given combat state.
- **Chat** ([code/botlib/be_ai_chat.c](code/botlib/be_ai_chat.c)) — a
  Markov-style template system that produces context-aware trash talk. Chat
  scripts are external `.c` files consumed by `l_precomp.c`, the in-tree C
  preprocessor.
- **Behavior FSM** — [code/game/ai_dmnet.c](code/game/ai_dmnet.c) — high-level
  state machine: "seek goal", "battle fight", "battle chase", "respawn",
  "intermission". Each state is a function that runs one frame and returns
  the next state's function pointer.

Read in this order: AAS structure → reachability → movement → goal → behavior
FSM. From the engine's perspective, a bot is a client whose input comes from
`BotAI()` instead of a network socket.

> **Concept.** Separating *navigation* (offline graph) from *deliberation*
> (runtime FSM) from *execution* (per-frame movement) is a clean three-layer
> architecture that recurs in every robotics stack from then to now.

---

## 12. The Toolchain

You are unlikely to *build* the toolchain — it is from 2005 and tied to gcc
2.95 — but you should understand its existence, because it is what makes the
QVM model possible.

### 12.1 LCC ([lcc/](lcc/))

The retargetable C compiler from Fraser & Hanson's textbook *A Retargetable C
Compiler: Design and Implementation* (Addison-Wesley, 1995). LCC is itself
worth a graduate course: it is a complete production C compiler that fits in
a small book and is extensively documented.

Source layout in [lcc/src/](lcc/src/):

| File | Role |
|---|---|
| [lcc/src/main.c](lcc/src/main.c) | Driver; orchestrates lex/parse/codegen |
| [lcc/src/lex.c](lcc/src/lex.c) | Lexer |
| [lcc/src/decl.c](lcc/src/decl.c) | Declaration parser (~31 KB — declaration grammar is the heaviest part of C) |
| [lcc/src/expr.c](lcc/src/expr.c) | Expression parser |
| [lcc/src/dag.c](lcc/src/dag.c) | DAG (directed acyclic graph) construction — IR |
| [lcc/src/gen.c](lcc/src/gen.c) | Code generation driver |
| [lcc/src/bytecode.c](lcc/src/bytecode.c) | The QVM backend that emits bytecode |
| [lcc/src/x86.md](lcc/src/x86.md), [alpha.md](lcc/src/alpha.md), [mips.md](lcc/src/mips.md) | Per-architecture machine descriptions in LCC's *iburg* tree-pattern DSL |

The pipeline:

```
C source
   │
   ▼  lex.c, decl.c, expr.c
parse tree
   │
   ▼  dag.c
DAG (LCC's internal IR — a forest of expression trees per basic block)
   │
   ▼  gen.c + iburg-generated tree-matcher from x86.md / bytecode emitter
target instructions (or bytecode)
```

The clever idea in LCC is that **code generation is driven by tree
pattern-matching** specified declaratively in a `.md` file. You write rules
like "if you see an ADD node with an integer constant on the right, emit
`ADD reg, imm`". An auxiliary tool called *iburg* turns the rules into a C
function that performs the match in linear time. To retarget LCC, you write a
new `.md` file and a small support library; you do not modify the core
compiler.

For QVM, the "machine description" is degenerate (the QVM is essentially a
stack machine), and most of the action is in
[lcc/src/bytecode.c](lcc/src/bytecode.c). It is one of the smaller backends
and a good first read.

> **Exercise.** Skim [lcc/src/x86.md](lcc/src/x86.md). Even without knowing
> the iburg DSL, you can recognize patterns like `addr: ADDP4(reg,acon)`. What
> kind of rule language is this? What problem is it solving that a hand-written
> emitter (like the one in `vm_x86.c`) handles by enumeration?

### 12.2 Q3ASM ([q3asm/](q3asm/))

A small assembler (~2000 lines of C) that consumes LCC's bytecode-target output
and produces `.qvm` files. Worth reading after you understand the bytecode
(§6) — it shows you what each opcode looks like at the source level. The
`.qvm` header (`vmHeader_t` in
[code/qcommon/qfiles.h](code/qcommon/qfiles.h)) records the magic number, code
length, data length, and lit length, which are all q3asm's job to compute.

### 12.3 Q3MAP ([q3map/](q3map/)) — covered in §13

### 12.4 Q3Radiant ([q3radiant/](q3radiant/))

The level editor. A Win32 MFC application — almost certainly not buildable on
a modern toolchain without significant porting. Skim, do not study.

### 12.5 BSPC ([code/bspc/](code/bspc/))

The bot navigation compiler. Reads a `.bsp` and produces a `.aas` file with
the area graph, reachabilities, and routing tables consumed by the bot
library at runtime.

---

## 13. The Offline Pipeline: q3map's BSP, VIS, and Light Stages

This is one of the most important and least-discussed parts of the codebase.
The runtime renderer is fast *because the offline tools did the work*.

### 13.1 Pipeline overview

[q3map/](q3map/) is the map compiler. Pipeline:

```
.map (text brushes, entities, patches)
    │
    ├── BSP        construct the binary space partition tree from brushes
    ├── PORTALS    place portals between BSP leafs
    ├── VIS        compute the per-leaf PVS by flooding through portals
    ├── LIGHT      bake static lightmaps (direct + radiosity)
    └── WRITE      emit final .bsp file
```

Driver: [q3map/bsp.c](q3map/bsp.c) (605 lines). Each stage reads the previous
stage's output and produces the next.

### 13.2 BSP construction

[q3map/bsp.c](q3map/bsp.c), [q3map/facebsp.c](q3map/facebsp.c),
[q3map/tree.c](q3map/tree.c): the map source is a list of *brushes* (convex
polyhedra defined by half-spaces). Construction recursively picks a splitting
plane at each step (typically the plane of one of the remaining brush faces),
partitions the remaining brushes into front/back lists, and recurses. Leaves
are convex regions of empty space.

Quality of the tree depends on the choice of splitter. A good splitter:
balances the front/back lists (keeps the tree shallow), splits as few brushes
as possible, and aligns with axis-aligned planes when possible (cheaper to
test). The heuristics are in `SelectSplitPlaneNum` in
[q3map/facebsp.c](q3map/facebsp.c).

### 13.3 Portals

[q3map/portals.c](q3map/portals.c): once the BSP is built, every leaf is a
convex region of empty space bounded by leaf-edges that are either solid (a
brush face) or transparent (the boundary between two empty leafs). The
transparent boundaries are *portals*. Each portal connects exactly two leafs.

[q3map/prtfile.c](q3map/prtfile.c) writes the portal list out to a `.prt` file
between the BSP and VIS stages.

### 13.4 VIS — exact visibility flooding

[q3map/vis.c](q3map/vis.c) (1197 lines) and
[q3map/visflow.c](q3map/visflow.c) compute, for every leaf L, the set of
other leafs that *might* be seen by some viewer standing in L. This is the
**Potentially Visible Set (PVS)**.

The algorithm — *Teller's portal-flow* — is conceptually simple and
operationally famous for being computationally brutal:

```
for each leaf L:
    for each portal P out of L:
        recursively flood through P:
            for each neighboring leaf N reached through P:
                for each portal P' out of N:
                    if a sight line from somewhere in L
                       can pass through both P and P':
                        mark N visible
                        recurse through P' into the next leaf
```

The "sight line passes through both P and P'" test is a 3D version of the
classic separating-hyperplane test: clip P' against the planes formed by L's
boundary and P, and if anything is left, the sight line exists. See
`PortalFlow` and the recursion in [q3map/visflow.c](q3map/visflow.c).

The cost is exponential in the worst case. A complex map (a few thousand
portals) could take *days* on a 1999 CPU. The output is a per-leaf bitvector
(one bit per other leaf), packed into the `.bsp` file as the `LUMP_VISIBILITY`
data. At runtime, the renderer simply loads the bitvector for the camera's leaf
and walks only the marked leafs. **The runtime PVS query is one bit-test per
leaf** — every cycle of expensive offline computation buys cycles of cheap
runtime culling.

> **Concept.** This is a classic offline/online tradeoff: an *amortized*
> precomputation that turns a hard runtime problem (visibility) into a trivial
> table lookup. Every modern occlusion-culling system (precomputed visibility
> portals in Unreal, baked occlusion in Unity, hierarchical Z-buffer
> occlusion) sits on the same shelf.

### 13.5 LIGHT and LIGHTV — radiosity baking

[q3map/light.c](q3map/light.c) (2149 lines) handles direct illumination:
shoot a ray from each light source to every lightmap texel and check
visibility (a `light_trace.c` ray cast against the BSP). Add the contribution
attenuated by distance and angle. Result: a lightmap texture per surface,
storing the static lighting as RGB.

[q3map/lightv.c](q3map/lightv.c) (5748 lines — the *largest* file in the
project) handles *radiosity*: indirect illumination, where light bounces off
diffuse surfaces and contributes to the lighting of nearby surfaces.

Algorithm sketch (this is form-factor radiosity):

1. Subdivide every surface into many small *patches* (one patch per lightmap
   texel, roughly).
2. Compute the *form factor* between every pair of patches — a geometric
   coefficient that measures how much one patch sees the other.
   `PointToPolygonFormFactor` (line 244 of `lightv.c`) does the per-pair work.
3. Iterate: each round, every patch shoots its accumulated light to every
   other patch, attenuated by the form factor. After a few rounds, the
   solution converges.

The cost is again huge — quadratic in the number of patches per round,
multiplied by ray-cast cost per pair. For a typical Q3 map (a few hundred
thousand patches) this is hours of CPU time. The output is folded into the
final lightmap texture, so the runtime cost is the same as direct-only
lighting: one texture read.

The rest of the file is implementation detail: surface subdivision, alpha
shading (light through transparent surfaces), back-face culling, color
bouncing, gamma correction.

### 13.6 Why this matters pedagogically

The runtime is fast because the offline pipeline did the hard work. This is a
recurring pattern in systems where startup cost is acceptable but per-frame
cost is not: compiled regular expressions, JIT'd code, ahead-of-time link-time
optimization, denormalized read-models in CQRS, materialized views in
databases, indexed Lucene segments. Q3 is an extreme example: *days* of
offline compute buy *milliseconds* of runtime cost.

> **Exercise.** Pick one of the offline stages (BSP, VIS, or LIGHTV) and
> estimate, for a hypothetical map with N=1000 brushes / N=2000 portals /
> N=200000 patches, the runtime cost of doing the work *online* per frame at
> 60 Hz. How big a CPU would you need? Express the answer in 2026 cores.

---

## 14. Cross-Cutting Themes

The recurring engineering ideas you should be able to articulate after
reading the codebase.

### 14.1 ABI as architecture

The engine/module split via the QVM ABI is the single most consequential
choice. It enables portability of mods, safety of mods, a stable interface for
an unbounded modding ecosystem, and testability/replaceability of modules.
The cost: every cross-boundary call goes through a switch and an integer
argument array. For a system that crosses the boundary thousands of times per
frame, this is significant.

### 14.2 Data-driven everything

Maps, models, shaders, sounds, fonts, menus, bot personalities, weapon
properties — all loaded from data files at runtime. The C source is a
*runtime* for content, not the content itself. This is now the universal
pattern in game engines and one of the under-appreciated reasons studios can
ship the games they ship.

### 14.3 Determinism as a feature

Because the same `bg_pmove.c` runs on server and client, *prediction works
only if the two produce identical output for identical input*. Any
nondeterminism (uninitialized memory, FP variation across CPUs,
random-number-state divergence) is a network bug. The whole codebase is
deterministic by construction: no `time(NULL)` in the simulation path, no
thread races on shared simulation state, no implicit globals that vary at
startup.

### 14.4 Snapshot-based state replication

Rather than streaming events between a stateful producer and a stateful
consumer (fragile under loss and reorder), publish periodic snapshots of
authoritative state and let the consumer reconcile. This pattern shows up in
distributed databases (Raft snapshots), CRDTs, configuration management
(Kubernetes desired-state reconciliation), and game networking.

### 14.5 Region-based memory

Game state has obvious lifetime tiers (per-level, per-frame, permanent).
Match the allocator to the tier and 90% of memory bugs become impossible to
represent.

### 14.6 Bit-budgeted protocols

The `entityStateFields[]` table in `msg.c` is the wire schema. Each field has
a chosen bit width — not the natural width of the host type. A 16-bit angle
is enough; an 8-bit health value is enough; a 5-bit weapon index is enough.
The discipline of *thinking in the protocol's units* is what made 28.8K
multiplayer feel responsive.

### 14.7 Defensive style without excessive defense

The engine is paranoid where it matters (filesystem, network, VM data
accesses, untrusted inputs) and trusting where it does not (its own data
structures, post-validation). `if ( !ent ) return;` at the top of many `g_*`
functions, but `assert`s rather than runtime checks deep in the renderer.
"Validate at boundaries, trust internally" is older than this codebase but
rarely executed this consistently.

### 14.8 Offline / online tradeoff

The most pervasive theme of all: VIS, lightmaps, AAS, lcc-compiled QVMs,
shader parsing — every expensive thing that *can* be done before the player
presses Start *is* done before the player presses Start.

---

## 15. Hands-On Exercises

The exercises are graded by effort. Pick three, do them deeply.

### Tier 1 — Reading

1. **Trace a bullet.** Press fire as a player. Walk the call chain from
   `usercmd_t` buttons → `bg_pmove` weapon state → `g_active.c` weapon firing
   → `g_weapon.c` `Bullet_Fire()` → `g_combat.c` damage → `entityState_t` →
   snapshot → `cl_parse.c` → `cgame` event → `cg_event.c`
   `EV_BULLET_HIT_WALL` → particle effect. Write a one-page diagram.
   *Tests*: cross-VM ABI understanding, networking boundaries.

2. **Read one snapshot byte-for-byte.** Patch the engine to log to a file the
   bytes of one outgoing snapshot, then hand-decode it using the
   `entityStateFields[]` table in [code/qcommon/msg.c](code/qcommon/msg.c).
   Verify the field-by-field delta encoding matches what the table describes.
   *Tests*: serialization, schema-driven encoding.

3. **Reverse-engineer the LCC bytecode for a tiny C function.** Write
   `int add(int a, int b) { return a+b; }` in `bg_lib.c`, build the QVM, and
   open the `.qvm` in a hex editor. Identify the `OP_ENTER`, `OP_LOCAL`,
   `OP_LOAD4`, `OP_ADD`, `OP_LEAVE` sequence. *Tests*: VM ISA, calling
   convention.

4. **Enumerate the syscalls.** Count the entries in `gameImport_t` in
   [code/game/g_public.h](code/game/g_public.h),
   `cgameImport_t` in [code/cgame/cg_public.h](code/cgame/cg_public.h), and
   `uiImport_t` in [code/q3_ui/ui_public.h](code/q3_ui/ui_public.h). Which
   syscalls are duplicated across modules? Why? *Tests*: ABI design.

### Tier 2 — Small modifications

5. **Add a console command.** Add a `say_loud` command to the server that
   broadcasts chat in uppercase. Identify exactly where to register it
   (hint: [code/game/g_cmds.c](code/game/g_cmds.c)) and what syscalls it
   needs. *Tests*: command system, configstring/command channel.

6. **Disable interpolation.** Find the entity interpolation path in
   [code/cgame/cg_ents.c](code/cgame/cg_ents.c) and short-circuit it.
   Observe the visual artifacts at 10 Hz snapshot rate vs 30 Hz. Now
   implement Catmull-Rom interpolation over 4 snapshots and compare. *Tests*:
   client prediction, snapshot timing.

7. **Swap the Huffman coder for raw transmission.** In
   [code/qcommon/net_chan.c](code/qcommon/net_chan.c), bypass `Huff_Compress`
   and `Huff_Decompress`. Measure bandwidth before and after on a one-bot
   match. *Tests*: compression ratio, real protocol overhead.

### Tier 3 — Deeper work

8. **Port `Q_rsqrt` to AVX.** Write a SIMD variant that computes 8 reciprocal
   square roots at once. Compare against `_mm256_rsqrt_ps` for accuracy and
   throughput. Then compare against `1.0f / sqrtf(x)` on a modern CPU; on
   most modern silicon the "obvious" version wins because Newton-Raphson
   single-precision RSQRT is a single-cycle op. Discuss why. *Tests*:
   numerical methods, micro-architecture awareness.

9. **Replace the netchan with QUIC.** Sketch the API changes required to move
   from raw UDP + reliable-command piggybacking to a QUIC stream. Identify
   which engine assumptions break (hint: think about which "unreliable"
   guarantees the engine *relies on*). *Tests*: protocol design.

10. **Write a tiny QVM disassembler.** Read a `.qvm` header, walk the
    bytecode, and print one line per instruction. Verify against
    `vm_interpreted.c`'s opcode handling. ~200 lines of Python. *Tests*: VM
    ISA, file format.

11. **Implement a minimal new game module.** Strip `game.qvm` down to a "tag"
    minigame: one team is "it"; touching transfers it. About 300 lines of C
    if you reuse the existing physics, item, and respawn code. *Tests*:
    end-to-end module development, the modder workflow.

12. **Write a fast VIS algorithm.** Replace q3map's portal-flow VIS with a
    stochastic ray-cast approximation: shoot N random rays from each leaf and
    record which other leafs they hit. Compare quality (how many false
    negatives — invisible leafs marked visible) at N=100, 1000, 10000 vs the
    exact algorithm. *Tests*: offline-pipeline understanding, sampling
    methods.

### Tier 4 — Synthesis

13. **Document the trust boundaries.** Produce a one-page taxonomy of every
    trust boundary in the system: VM↔engine, engine↔network,
    engine↔filesystem, engine↔OS. For each, identify what is being trusted,
    what is being checked, and where in the code. *Tests*: security mindset.

14. **Compare and contrast with WebAssembly.** Write a 1500-word essay on the
    similarities and differences between QVM and Wasm. Cover: memory model,
    type system, control flow, module/import structure, JIT compilation
    strategy, host trust model. *Tests*: ability to relate historical and
    modern systems.

---

## 16. Further Reading

- *A Retargetable C Compiler: Design and Implementation*, Fraser & Hanson —
  the LCC textbook. The compiler in [lcc/](lcc/) is the reference
  implementation.
- *Game Engine Architecture* (3rd ed.), Jason Gregory — modern context for
  what Q3 pioneered: subsystem layering, resource management, real-time loops.
- Foundations of Multithreaded, Parallel, and Distributed Programming,
  Andrews — the classical theory behind snapshot-based replication.
- "Tribes Networking Model" (Frohnmayer & Gift, 2000) — companion paper to
  the Q3 network architecture; describes a sibling design that uses
  event-stream replication instead of snapshots, with explicit comparison.
- Fabien Sanglard's *Quake III Source Code Review* — outstanding annotated
  tour at a more code-bound level than this guide.
- Bret Jackson's writeup on *The Quake 3 Networking Model* — close in spirit
  to §7 here but with packet captures.
- Teller, S. *Visibility Computations in Densely Occluded Polyhedral
  Environments*, PhD thesis, UC Berkeley, 1992 — the original work behind
  Q3's VIS algorithm.
- Cohen & Greenberg, *The Hemi-Cube: A Radiosity Solution for Complex
  Environments* (SIGGRAPH 1985) — the radiosity tradition q3map's `lightv.c`
  belongs to.
- *id Tech 3* on Wikipedia — historical context (forks, ioquake3, Tremulous,
  Urban Terror, World of Padman).

---

## 17. Reference: Where to Find What

| Topic | File |
|---|---|
| `vec3_t`, `playerState_t`, `entityState_t`, `usercmd_t` | [code/game/q_shared.h](code/game/q_shared.h) |
| Vector math, `Q_rsqrt` | [code/game/q_math.c](code/game/q_math.c) |
| `Com_Init`, main loop, allocators | [code/qcommon/common.c](code/qcommon/common.c) |
| Console commands | [code/qcommon/cmd.c](code/qcommon/cmd.c) |
| Console variables | [code/qcommon/cvar.c](code/qcommon/cvar.c) |
| Virtual filesystem and `.pk3` overlay | [code/qcommon/files.c](code/qcommon/files.c) |
| Bit-packed wire I/O, delta encoders | [code/qcommon/msg.c](code/qcommon/msg.c) |
| Reliable UDP channel + fragmentation | [code/qcommon/net_chan.c](code/qcommon/net_chan.c) |
| Adaptive Huffman | [code/qcommon/huffman.c](code/qcommon/huffman.c) |
| QVM loader and dispatcher | [code/qcommon/vm.c](code/qcommon/vm.c) |
| QVM interpreter (the spec) | [code/qcommon/vm_interpreted.c](code/qcommon/vm_interpreted.c) |
| QVM x86 JIT | [code/qcommon/vm_x86.c](code/qcommon/vm_x86.c) |
| Collision (BSP traversal, brush clipping) | [code/qcommon/cm_trace.c](code/qcommon/cm_trace.c) |
| Curved-surface collision | [code/qcommon/cm_patch.c](code/qcommon/cm_patch.c) |
| Server snapshot building | [code/server/sv_snapshot.c](code/server/sv_snapshot.c) |
| Server game-syscall dispatch | [code/server/sv_game.c](code/server/sv_game.c) |
| Server main loop | [code/server/sv_main.c](code/server/sv_main.c) |
| Client snapshot parsing | [code/client/cl_parse.c](code/client/cl_parse.c) |
| Client cgame-syscall dispatch | [code/client/cl_cgame.c](code/client/cl_cgame.c) |
| Client input → usercmd | [code/client/cl_input.c](code/client/cl_input.c) |
| Game module entry points | [code/game/g_main.c](code/game/g_main.c) |
| Game syscall enum (the ABI) | [code/game/g_public.h](code/game/g_public.h) |
| Game syscall wrappers | [code/game/g_syscalls.c](code/game/g_syscalls.c) |
| Player movement (server + client prediction) | [code/game/bg_pmove.c](code/game/bg_pmove.c) |
| Trajectory evaluation, item table | [code/game/bg_misc.c](code/game/bg_misc.c) |
| Cgame entry points | [code/cgame/cg_main.c](code/cgame/cg_main.c) |
| Cgame syscall enum | [code/cgame/cg_public.h](code/cgame/cg_public.h) |
| Client-side prediction | [code/cgame/cg_predict.c](code/cgame/cg_predict.c) |
| Renderer frontend | [code/renderer/tr_main.c](code/renderer/tr_main.c) |
| Renderer backend | [code/renderer/tr_backend.c](code/renderer/tr_backend.c) |
| Shader script parser | [code/renderer/tr_shader.c](code/renderer/tr_shader.c) |
| BSP loader and PVS | [code/renderer/tr_bsp.c](code/renderer/tr_bsp.c), [code/renderer/tr_world.c](code/renderer/tr_world.c) |
| MD3 model rendering | [code/renderer/tr_mesh.c](code/renderer/tr_mesh.c) |
| Bezier patch tessellation | [code/renderer/tr_curve.c](code/renderer/tr_curve.c) |
| Sound DMA / mixer driver | [code/client/snd_dma.c](code/client/snd_dma.c) |
| Sound mixer kernels | [code/client/snd_mix.c](code/client/snd_mix.c) |
| ADPCM decoder | [code/client/snd_adpcm.c](code/client/snd_adpcm.c) |
| Bot AAS pathfinding | [code/botlib/be_aas_route.c](code/botlib/be_aas_route.c) |
| Bot deathmatch FSM | [code/game/ai_dmnet.c](code/game/ai_dmnet.c) |
| Bot weapon selection | [code/botlib/be_ai_weap.c](code/botlib/be_ai_weap.c) |
| Bot chat AI | [code/botlib/be_ai_chat.c](code/botlib/be_ai_chat.c) |
| Bot navigation compiler | [code/bspc/](code/bspc/) |
| LCC C compiler | [lcc/](lcc/) |
| LCC QVM backend | [lcc/src/bytecode.c](lcc/src/bytecode.c) |
| QVM assembler | [q3asm/](q3asm/) |
| Map compiler driver | [q3map/bsp.c](q3map/bsp.c) |
| BSP construction | [q3map/facebsp.c](q3map/facebsp.c), [q3map/tree.c](q3map/tree.c) |
| Portals and PRT file | [q3map/portals.c](q3map/portals.c), [q3map/prtfile.c](q3map/prtfile.c) |
| VIS (PVS computation) | [q3map/vis.c](q3map/vis.c), [q3map/visflow.c](q3map/visflow.c) |
| Direct lighting | [q3map/light.c](q3map/light.c) |
| Radiosity | [q3map/lightv.c](q3map/lightv.c) |

---

*End of guide. Total: ~9,500 words. Estimated reading time: 60 minutes for the
guide itself, 25–35 hours for the recommended source-code journey including
the offline pipeline.*
