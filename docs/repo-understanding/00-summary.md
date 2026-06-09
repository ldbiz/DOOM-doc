# 00 - BSP Summary

## What role does BSP partitioning play in this repo?

BSP (Binary Space Partitioning) serves two distinct purposes in the linuxdoom-1.10 codebase:

1. **Rendering traversal** — The core renderer walks the BSP tree front-to-back relative to the viewpoint each frame. This front-to-back ordering enables efficient occlusion clipping: nearer walls are processed first, filling clip arrays that cull farther geometry. The BSP entirely determines wall rendering order.

2. **Line-of-sight visibility** — `P_CheckSight()` uses the BSP tree (after a fast REJECT-table pre-check) to determine whether two map objects can see each other. It traverses the BSP, testing each subsector's segs to detect occlusion.

BSP is **not** used for collision detection/movement. That role belongs to the BLOCKMAP (a 2D spatial hash grid), implemented via `P_BlockLinesIterator()` / `P_BlockThingsIterator()` in `p_maputl.c`.

## Main BSP-related data structures

| Structure | File | Role |
|---|---|---|
| `node_t` | `r_defs.h` | Runtime BSP node: partition line (x,y,dx,dy), child bbox[2][4], children[2] |
| `mapnode_t` | `doomdata.h` | On-disk (WAD) BSP node: 16-bit versions of the above |
| `subsector_t` | `r_defs.h` | Runtime subsector: sector pointer, seg count (`numlines`) and index (`firstline`) |
| `mapsubsector_t` | `doomdata.h` | On-disk subsector: `numsegs`, `firstseg` |
| `seg_t` | `r_defs.h` | Runtime line segment: vertices, linedef/sidedef pointers, front/back sector pointers, angle, offset |
| `mapseg_t` | `doomdata.h` | On-disk seg: 16-bit vertex/linedef/side indices, angle, offset |
| `vertex_t` | `r_defs.h` | Runtime vertex: fixed-point x, y |
| `drawseg_t` | `r_defs.h` | Per-frame rendered wall segment: screen x range, scale values, silhouette info, sprite-clip pointers |
| `visplane_t` | `r_defs.h` | Per-frame floor/ceiling span accumulator: height, picnum, lightlevel, per-column top/bottom arrays |
| `vissprite_t` | `r_defs.h` | Per-frame projected sprite: screen range, scale, texture, colormap |

## Main BSP-related files

| Module | Primary files |
|---|---|
| **BSP data rendering traversal** | `r_bsp.c`, `r_bsp.h` |
| **Seg clipping & drawseg creation** | `r_segs.c`, `r_segs.h` |
| **Floor/ceiling visplane batching** | `r_plane.c`, `r_plane.h` |
| **Sprite collection & masked drawing** | `r_things.c`, `r_things.h` |
| **Rendering entry point / view setup** | `r_main.c`, `r_main.h` |
| **Map/WAD lump loading** | `p_setup.c` |
| **Line-of-sight (BSP-based)** | `p_sight.c` |
| **WAD file access** | `w_wad.c`, `w_wad.h` |
| **Runtime type definitions** | `r_defs.h`, `r_state.h` |
| **On-disk type definitions** | `doomdata.h` |
| **Fixed-point math** | `m_fixed.h` |
| **Bounding box utilities** | `m_bbox.c`, `m_bbox.h` |

## Main runtime flow: WAD data → BSP usage

1. **WAD loading** (`w_wad.c`): `W_InitMultipleFiles()` reads WAD directory, builds `lumpinfo[]` table.
2. **Map lump loading** (`p_setup.c`): `P_SetupLevel()` loads lumps in order — `P_LoadBlockMap()`, `P_LoadVertexes()`, `P_LoadSectors()`, `P_LoadSideDefs()`, `P_LoadLineDefs()`, `P_LoadSubsectors()`, `P_LoadNodes()`, `P_LoadSegs()`.
3. **Conversion**: Each loader casts raw lump bytes to `map*_t` structs (16-bit fields), converts via `SHORT()` and `<<FRACBITS` into runtime `*_t` types (32-bit fixed-point), and stores in global arrays (`nodes[]`, `subsectors[]`, `segs[]`, etc.).
4. **Per-frame rendering** (`r_main.c` → `r_bsp.c`): `R_RenderPlayerView()` calls `R_RenderBSPNode(numnodes-1)` which recursively traverses, calling `R_Subsector()` at leaves to process segs, visplanes, and sprites.
5. **Per-frame LOS** (`p_sight.c`): `P_CheckSight()` checks REJECT table, then traverses BSP via `P_CrossBSPNode()` to test seg occlusion.

## BSP usage domains

- ✅ **Rendering** — Wall ordering, occlusion clipping, visplane collection
- ✅ **Visibility (LOS)** — Line-of-sight checks for monster AI
- ❌ **Collision** — Uses BLOCKMAP instead
- ❌ **Movement** — Uses BLOCKMAP instead
- ✅ **Map loading** — BSP data loaded from WAD NODES/SSECTORS/SEGS lumps

## Read these first (priority order)

1. `linuxdoom-1.10/doomdata.h` — On-disk map structures, `NF_SUBSECTOR`, map lump enum
2. `linuxdoom-1.10/r_defs.h` — Runtime structures: `node_t`, `subsector_t`, `seg_t`, `drawseg_t`, `visplane_t`
3. `linuxdoom-1.10/p_setup.c` — `P_LoadNodes()`, `P_LoadSubsectors()`, `P_LoadSegs()` — how BSP loads from WAD
4. `linuxdoom-1.10/r_bsp.c` — `R_RenderBSPNode()`, `R_Subsector()`, `R_AddLine()` — core traversal
5. `linuxdoom-1.10/r_segs.c` — `R_AddLine()`, `R_StoreWallRange()`, clipping — seg-to-drawseg pipeline
6. `linuxdoom-1.10/r_plane.c` — `R_FindPlane()`, `R_CheckPlane()`, visplane mgmt
7. `linuxdoom-1.10/r_main.c` — `R_RenderPlayerView()`, `R_PointOnSide()`, `R_PointInSubsector()`
8. `linuxdoom-1.10/p_sight.c` — `P_CheckSight()`, `P_CrossBSPNode()` — BSP for visibility
9. `linuxdoom-1.10/r_things.c` — `R_AddSprites()`, sprite collection during subsector processing
10. `linuxdoom-1.10/m_fixed.h` — `FRACBITS`, `FRACUNIT`, fixed-point coordinate system
