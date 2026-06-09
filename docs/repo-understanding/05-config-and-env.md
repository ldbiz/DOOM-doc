# 05 - BSP Configuration & Environment

## Summary

There is no explicit BSP-specific configuration in this codebase. BSP behaviour is entirely determined by:

- **Map/WAD data** — The BSP tree structure comes from the WAD file; the code has no runtime BSP generation or modification.
- **Hardcoded constants** — Limits, lump naming, coordinate scaling, and screen dimensions are all compile-time constants.
- **Platform defines** — The `-DLINUX -DNORMALUNIX` flags in the Makefile control endian handling, but BSP logic itself is platform-independent.

## Map/WAD selection

Maps are selected at runtime via command-line arguments or game logic (not BSP-specific). The map name (e.g., `"E1M1"`) is passed to `P_SetupLevel()` which locates the starting lump index via `W_GetNumForName()`. All BSP data for the map is loaded from lumps at fixed offsets from that index.

## Hardcoded assumptions about map format

| Assumption | Where | Detail |
|---|---|---|
| Lump naming convention | `doomdata.h` (`ML_*` enum) | Map lumps have fixed names: `THINGS`, `LINEDEFS`, `SIDEDEFS`, `VERTEXES`, `SEGS`, `SSECTORS`, `NODES`, `SECTORS`, `REJECT`, `BLOCKMAP` |
| Lump loading order | `p_setup.c` (`P_SetupLevel()`) | Must be loaded in order: BLOCKMAP → VERTEXES → SECTORS → SIDEDEFS → LINEDEFS → SSECTORS → NODES → SEGS |
| Coordinate scale | `m_fixed.h` | `FRACBITS = 16` — all map coordinates are stored as 16-bit shorts in WAD, scaled to 32-bit fixed-point on load |
| Endianness | `m_swap.h` | WAD data is little-endian; `SHORT()`/`LONG()` macros handle big-endian platforms |
| Child node encoding | `doomdata.h` | `NF_SUBSECTOR = 0x8000` — 16-bit children, high bit marks subsector leaves |
| Root node convention | `r_main.c`, `r_bsp.c` | BSP root is always at `nodes[numnodes - 1]` |
| Lump cache tag | `p_setup.c` | `PU_STATIC` for temporary loads (freed after conversion), `PU_LEVEL` for persistent (BLOCKMAP, REJECT) |

## Runtime flags that affect BSP-adjacent behaviour

There are no runtime flags that specifically control BSP traversal. However, these compile-time constants affect BSP-related rendering and clipping:

| Constant | Value | File | BSP relevance |
|---|---|---|---|
| `SCREENWIDTH` | `320` | `doomdef.h` | Sizes visplane `top[]`/`bottom[]` arrays and `floorclip[]`/`ceilingclip[]` arrays |
| `SCREENHEIGHT` | `200` | `doomdef.h` | Clipping bounds |
| `MAXDRAWSEGS` | `256` | `r_defs.h` | Max wallpaper drawsegs per frame — overflow causes `I_Error` |
| `MAXVISPLANES` | `128` | `r_plane.c` | Max floor/ceiling visplanes per frame — overflow causes `I_Error` |
| `MAXVISSPRITES` | `128` | `r_things.h` | Max projected sprites per frame |
| `MAXSEGS` | `32` | `r_bsp.c` | Max solid clip ranges during wall rendering |
| `MAXOPENINGS` | `SCREENWIDTH*64` | `r_plane.c` | Size of openings buffer for sprite clipping data |

## Compile-time defines (Makefile)

```
CFLAGS = -g -Wall -DNORMALUNIX -DLINUX
```

- `-DLINUX`: Enables Linux-specific code paths (X11 video, sound server, timer).
- `-DNORMALUNIX`: Enables standard Unix behaviour (no DOS-specific code).
- Neither define directly affects BSP logic. Endian handling is controlled by `__BIG_ENDIAN__` detection in `m_swap.h`.

## Test fixture configuration

There are no BSP-specific test fixtures or test data in this repo. The only way to exercise BSP behaviour is to run the full application with a DOOM WAD file. See `06-tests-and-fixtures.md` for details.

## What is data-driven vs hardcoded

- **Data-driven**: The BSP tree topology (node count, subsector count, seg layout, partition line positions) comes entirely from the WAD file. Map authors and BSP node-builders determine this.
- **Hardcoded**: All limits (`MAXDRAWSEGS`, `MAXVISPLANES`, etc.), the coordinate system (`FRACBITS=16`), the `NF_SUBSECTOR` leaf-encoding scheme, the screen resolution, and the lump naming convention.
