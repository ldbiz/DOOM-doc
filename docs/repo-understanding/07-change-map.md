# 07 - BSP Change Map

## If I need to change map/WAD lump loading

**Likely files**: `linuxdoom-1.10/p_setup.c`, `linuxdoom-1.10/doomdata.h`, `linuxdoom-1.10/w_wad.c`, `linuxdoom-1.10/m_swap.h`

| File | Why it matters |
|---|---|
| `p_setup.c` | All `P_Load*()` functions live here. To change lump loading order, add new lump types, or modify conversion logic, this is the primary file. |
| `doomdata.h` | On-disk map types (`mapnode_t`, `mapsubsector_t`, `mapseg_t`). If the WAD format changes, update these structs and the `ML_*` enum. |
| `w_wad.c` | `W_CacheLumpNum()`, `W_GetNumForName()` — if lump lookup or caching behaviour needs to change. |
| `m_swap.h` | `SHORT()`/`LONG()` macros — if endian handling or field sizes change. |

**Caveats**:
- Lump loading order IS significant. `P_LoadLineDefs()` depends on `P_LoadVertexes()` and `P_LoadSideDefs()` already having run. `P_LoadSegs()` depends on everything before it.
- All loaders use `Z_Malloc(PU_LEVEL, ...)` — memory is freed on level change via zone tag purging.
- `P_GroupLines()` must run after line/sector/subsector loading and before things are spawned.

---

## If I need to change BSP node/subsector representation

**Likely files**: `linuxdoom-1.10/r_defs.h`, `linuxdoom-1.10/doomdata.h`, `linuxdoom-1.10/p_setup.c`

| File | Why it matters |
|---|---|
| `r_defs.h` | Runtime struct definitions: `node_t`, `subsector_t`, `seg_t`. Change fields here to alter the in-memory representation. |
| `doomdata.h` | On-disk struct definitions and `NF_SUBSECTOR`. Change these if the WAD binary format changes too. |
| `p_setup.c` | `P_LoadNodes()`, `P_LoadSubsectors()`, `P_LoadSegs()` — the conversion functions that populate runtime structs from WAD data. Must be updated to match any struct changes. |

**Caveats**:
- `NF_SUBSECTOR = 0x8000` is a 16-bit convention. If you widen `children[]` beyond 16 bits, the subsector/node discrimination logic in `r_bsp.c`, `r_main.c`, and `p_sight.c` must all change.
- `node_t` uses `fixed_t` (32-bit) for coordinates; `mapnode_t` uses `short` (16-bit). The conversion in `P_LoadNodes()` applies `<<FRACBITS` scaling. Changing `FRACBITS` affects coordinate precision globally.
- Subsector seg count/index are `short` — max 32767 segs per subsector.

---

## If I need to change BSP traversal order

**Likely files**: `linuxdoom-1.10/r_bsp.c`, `linuxdoom-1.10/r_main.c`

| File | Why it matters |
|---|---|
| `r_bsp.c` | `R_RenderBSPNode()` — the recursive traversal function. The order of recursive calls determines front-to-back vs back-to-front traversal. Currently: front child first, then back child if bbox visible. |
| `r_main.c` | `R_PointOnSide()` — the partition-line side test. Changing the side-determination math changes which child is considered "front". |

**Caveats**:
- The current front-to-back order is fundamental to the solidseg occlusion clipping system. Reversing it would break wall rendering (far walls would overwrite near walls).
- `solidsegs[]` (cliprange_t) assumes front-to-back: earlier segments block later ones. Changing traversal order requires rethinking the clip system.
- `R_CheckBBox()` frustum culling in the back-child path is an optimization — skipping it is safe but slower. Making it too aggressive could cause missing geometry.

---

## If I need to change wall rendering from BSP traversal

**Likely files**: `linuxdoom-1.10/r_segs.c`, `linuxdoom-1.10/r_bsp.c`, `linuxdoom-1.10/r_draw.c`

| File | Why it matters |
|---|---|
| `r_segs.c` | `R_AddLine()`, `R_ClipSolidWallSegment()`, `R_ClipPassWallSegment()`, `R_StoreWallRange()`, `R_RenderSegLoop()` — the entire seg-to-pixels pipeline. |
| `r_bsp.c` | `R_Subsector()` is where seg iteration starts; `R_AddLine()` dispatch logic is here. |
| `r_draw.c` | `R_DrawColumn()` — the low-level column-drawing primitive called by `R_RenderSegLoop()`. |

**Caveats**:
- `R_StoreWallRange()` creates `drawseg_t` entries that are later consumed by sprite rendering (`r_things.c`). Any changes to drawseg fields (especially `sprtopclip`, `sprbottomclip`, `maskedtexturecol`, `silhouette`) must stay compatible with `R_DrawSprite()`.
- `solidsegs[]` clip-range list modification in `R_ClipSolidWallSegment()` is delicate — off-by-one errors cause visual artifacts or crashes.
- Per-column `floorclip[]`/`ceilingclip[]` updates in `R_RenderSegLoop()` affect both visplane span generation and sprite clipping.

---

## If I need to change visibility/clipping behaviour

**Likely files**: `linuxdoom-1.10/r_bsp.c`, `linuxdoom-1.10/r_segs.c`, `linuxdoom-1.10/r_plane.c`, `linuxdoom-1.10/p_sight.c`

| File | Why it matters |
|---|---|
| `r_bsp.c` | `R_CheckBBox()` — frustum culling of back children during traversal. |
| `r_segs.c` | `R_ClipSolidWallSegment()`, `R_ClipPassWallSegment()` — screen-x clipping against solidsegs. |
| `r_plane.c` | `R_FindPlane()`, `R_CheckPlane()`, `floorclip[]`/`ceilingclip[]` — visplane merging and vertical clip tracking. |
| `p_sight.c` | `P_CheckSight()`, `P_CrossBSPNode()` — line-of-sight visibility. |

**Caveats**:
- Visplane merging in `R_CheckPlane()` depends on `top[]` entries being `0xff` (unset). Changing the sentinel value or the merge logic can cause floor/ceiling rendering errors.
- `solidsegs[]` is a fixed-size array (`MAXSEGS = 32`). Complex scenes can exhaust it.
- LOS uses the REJECT table first — changing BSP traversal for LOS without considering REJECT interactions could break monster AI.

---

## If I need to change collision or spatial queries

**Likely files**: `linuxdoom-1.10/p_maputl.c`, `linuxdoom-1.10/p_map.c`, `linuxdoom-1.10/p_local.h`

| File | Why it matters |
|---|---|
| `p_maputl.c` | `P_BlockLinesIterator()`, `P_BlockThingsIterator()`, `P_PathTraverse()` — BLOCKMAP-based spatial queries. Also `P_SetThingPosition()` which uses `R_PointInSubsector()`. |
| `p_map.c` | `P_CheckPosition()`, `P_TryMove()` — collision entry points that use blockmap iterators. |
| `p_local.h` | `MAPBLOCKUNITS`, `MAPBLOCKSIZE`, `PLAYERRADIUS` — blockmap grid and collision radius constants. |

**Caveats**:
- Collision does NOT use BSP. It uses BLOCKMAP. If you want to add BSP-based collision, you'd need to either replace or supplement the blockmap system.
- `R_PointInSubsector()` (BSP-based point lookup) is used by `P_SetThingPosition()` to link mobjs to subsectors for rendering and sound propagation — not for collision.

---

## If I need to add BSP debug visualisation

**Likely files**: `linuxdoom-1.10/r_bsp.c`, `linuxdoom-1.10/r_main.c`, `linuxdoom-1.10/am_map.c`

| File | Why it matters |
|---|---|
| `r_bsp.c` | Instrument `R_RenderBSPNode()` to log node indices, partition lines, and traversal decisions. |
| `r_main.c` | `R_PointInSubsector()` can be instrumented to verify point→subsector resolution. |
| `am_map.c` | The automap already draws linedefs. Could be extended to overlay BSP partition lines, subsector boundaries, or node bounding boxes. |

**Caveats**:
- Adding real-time BSP visualization during gameplay is expensive. The BSP has many nodes; rendering all partition lines every frame would hurt performance.
- A separate debug dump tool that processes the loaded BSP arrays would be safer than modifying the renderer.

---

## If I need to add or update BSP tests/fixtures

**Likely files**: (none exist yet — would need to be created)

**Key areas to instrument**:
- `p_setup.c`: Verify `P_LoadNodes()`, `P_LoadSubsectors()`, `P_LoadSegs()` output against known-good data.
- `r_main.c`: Test `R_PointOnSide()` with known partition lines and points.
- `r_main.c`: Test `R_PointInSubsector()` with known world coordinates.
- `p_sight.c`: Test `P_CheckSight()` with known-visible and known-occluded mobj pairs.

**Caveats**:
- No test framework exists. Would need to add one or write standalone test programs.
- No test WAD files included. Tests would need a minimal hand-crafted WAD or a mock BSP tree constructed in code.
- BSP traversal is tightly coupled to the rendering pipeline — isolating BSP logic for unit testing requires significant refactoring.
