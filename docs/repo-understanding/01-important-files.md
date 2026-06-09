# 01 - Important BSP-Related Files

Files are grouped by BSP subsystem. Only files relevant to BSP understanding are listed.

## Map/WAD lump loading

| File | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/doomdata.h` | On-disk map type definitions (`mapnode_t`, `mapsubsector_t`, `mapseg_t`, `mapvertex_t`), `NF_SUBSECTOR` constant, map lump enum (`ML_NODES`, `ML_SSECTORS`, `ML_SEGS`, etc.) | Defines the WAD binary format for BSP data; `NF_SUBSECTOR = 0x8000` is the critical leaf-node marker |
| `linuxdoom-1.10/p_setup.c` | `P_LoadNodes()`, `P_LoadSubsectors()`, `P_LoadSegs()`, `P_SetupLevel()`, global BSP array definitions (`nodes[]`, `subsectors[]`, `segs[]`) | All BSP data flows through here: converts 16-bit WAD structs → 32-bit fixed-point runtime structs |
| `linuxdoom-1.10/w_wad.c` | `W_GetNumForName()`, `W_CacheLumpNum()`, `W_CacheLumpName()`, `W_LumpLength()` | How map lumps are located in WAD files and cached into memory |
| `linuxdoom-1.10/w_wad.h` | `lumpinfo_t`, `lumpcache[]`, `numlumps` | WAD directory and cache structures |
| `linuxdoom-1.10/m_swap.h` | `SHORT()`, `LONG()` endian macros | Ensures correct byte-order when reading 16/32-bit WAD fields |
| `linuxdoom-1.10/m_fixed.h` | `FRACBITS` (16), `FRACUNIT` (1<<16), `fixed_t` | Coordinate scaling: WAD shorts are shifted left 16 bits to become fixed-point runtime coordinates |

## BSP data structures

| File | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/r_defs.h` | Runtime `node_t`, `subsector_t`, `seg_t`, `vertex_t`, `line_t`, `side_t`, `sector_t` definitions | Complete runtime BSP data model; all fields used during traversal and rendering |
| `linuxdoom-1.10/r_state.h` | `extern` declarations for `numvertexes/vertexes`, `numsegs/segs`, `numsubsectors/subsectors`, `numnodes/nodes`, `numlines/lines`, `numsides/sides`; view globals (`viewx`, `viewy`, `viewz`, `viewangle`) | Shared renderer state — the view point used for BSP traversal side-determination |
| `linuxdoom-1.10/p_mobj.h` | `mobj_t` definition including `subsector` pointer and blockmap links | Shows how map objects (things) are linked to subsectors for spatial queries |

## BSP traversal

| File | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/r_bsp.c` | `R_RenderBSPNode()` (recursive traversal), `R_Subsector()` (leaf processing), `R_AddLine()` (seg→drawseg dispatch), `R_CheckBBox()` (frustum cull), drawsegs array definition | **The single most important BSP file.** Contains the complete rendering traversal: front-to-back recursion, NF_SUBSECTOR leaf detection, subsector processing dispatch |
| `linuxdoom-1.10/r_bsp.h` | Externs for `drawsegs[]`, `ds_p`, `curline`, `sidedef`, `linedef`, `frontsector`, `backsector` | Interface between BSP traversal and seg/plane/sprite code |
| `linuxdoom-1.10/r_main.c` | `R_RenderPlayerView()` (frame entry), `R_PointOnSide()` (partition side test), `R_PointInSubsector()` (point→subsector lookup via BSP), `R_SetupFrame()` (view setup) | Initiates BSP traversal each frame; contains the partition-line side-test math |
| `linuxdoom-1.10/r_main.h` | `R_PointOnSide()`, `R_PointInSubsector()` prototypes | |
| `linuxdoom-1.10/p_sight.c` | `P_CheckSight()`, `P_CrossBSPNode()`, `P_CrossSubsector()` | BSP-based line-of-sight: traverses BSP to test seg occlusion between two mobjs |
| `linuxdoom-1.10/m_bbox.c` | `M_ClearBox()`, `M_AddToBox()` | Bounding box math used by node bbox checks (`R_CheckBBox`) and collision |

## Rendering / visibility

| File | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/r_segs.c` | `R_StoreWallRange()` (drawseg creation), `R_ClipSolidWallSegment()`, `R_ClipPassWallSegment()` (screen-x clipping), `R_RenderSegLoop()` (per-column wall drawing) | Converts segs from BSP traversal into drawsegs with scale, texture, and clipping info. Manages the `solidsegs[]` clip-range list. |
| `linuxdoom-1.10/r_segs.h` | `R_RenderMaskedSegRange()` declaration | |
| `linuxdoom-1.10/r_plane.c` | `R_FindPlane()`, `R_CheckPlane()` (visplane merging), `R_DrawPlanes()` (batch draw), `visplanes[]` array, `floorclip[]`/`ceilingclip[]` arrays, `openings[]` buffer | Floor/ceiling spans are accumulated into visplanes during BSP traversal, then drawn in bulk. Per-column clipping arrays track what's already been drawn. |
| `linuxdoom-1.10/r_plane.h` | Visplane and clip array externs | |
| `linuxdoom-1.10/r_things.c` | `R_AddSprites()` (called from `R_Subsector`), `R_ProjectSprite()` (world→screen projection), `R_DrawSprite()`, `R_DrawMasked()`, `vissprites[]` | Sprites are collected per-subsector during BSP traversal, sorted by distance, and drawn after visplanes with per-column clipping against drawsegs |
| `linuxdoom-1.10/r_things.h` | `MAXVISSPRITES`, vissprite externs | |
| `linuxdoom-1.10/r_draw.c` | `R_DrawColumn()`, `R_DrawSpan()`, `dc_*`/`ds_*` globals | Low-level column/span drawing primitives used by seg and plane rendering |
| `linuxdoom-1.10/r_sky.c` | Sky rendering | Sky flats interact with BSP visplane handling (special-cased in `R_FindPlane`) |

## Collision / spatial queries

| File | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/p_maputl.c` | `P_BlockLinesIterator()`, `P_BlockThingsIterator()`, `P_PathTraverse()`, `P_SetThingPosition()`/`P_UnsetThingPosition()` | Collision uses BLOCKMAP (not BSP). Included here to clarify the separation: BSP is for rendering/LOS, BLOCKMAP is for movement/collision. |
| `linuxdoom-1.10/p_map.c` | `P_CheckPosition()`, `P_TryMove()`, `P_SlideMove()` | High-level collision that uses BLOCKMAP iterators |
| `linuxdoom-1.10/p_local.h` | `MAXINTERCEPTS`, `MAPBLOCKUNITS`, `MAPBLOCKSIZE`, `PLAYERRADIUS` | Constants relevant to spatial queries and coordinate scaling |

## Configuration / build

| File | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/Makefile` | Compile flags: `-DNORMALUNIX -DLINUX` | Platform defines affect endian handling and memory allocation |
| `linuxdoom-1.10/doomdef.h` | `SCREENWIDTH` (320), `SCREENHEIGHT` (200) | Screen dimensions determine visplane top/bottom array sizes, floorclip/ceilingclip sizes, and `MAXOPENINGS` (SCREENWIDTH*64) |
