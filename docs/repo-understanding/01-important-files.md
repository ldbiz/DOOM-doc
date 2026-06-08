# Important Files for BSP Understanding

## Map/WAD/Lump Loading

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/w_wad.c` / `w_wad.h` | WAD file I/O; reads lumps from disk | All map data (including BSP lumps) passes through W_CacheLumpNum() and W_ReadLump(). Entry point for all BSP data into the system. |
| `linuxdoom-1.10/p_setup.c` | Deserializes all map lumps; orchestrates loading | Contains P_LoadVertexes, P_LoadSegs, P_LoadNodes, P_LoadSubsectors, P_LoadSectors, P_LoadLineDefs, P_LoadSideDefs. Converts WAD format to runtime format. Critical to understanding BSP initialization. |
| `linuxdoom-1.10/doomdata.h` | Defines WAD on-disk format | mapvertex_t, mapnode_t, mapsubsector_t, mapseg_t, maplinedef_t, mapsidedef_t, mapsector_t. Required to understand the binary layout of BSP data in WAD files. |

## BSP Data Structures

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/r_defs.h` | Defines runtime BSP data structures | node_t, seg_t, subsector_t, vertex_t, line_t, side_t, sector_t, drawseg_t. These are the in-memory representations of BSP and map data. |
| `linuxdoom-1.10/doomdef.h` | Defines constants and high-level types | Includes #defines for MAXDRAWSEGS, coordinate limits, game constants. Needed for understanding global limits. |
| `linuxdoom-1.10/m_fixed.h` | Fixed-point math definitions | Defines fixed_t, FRACBITS, FRACUNIT, FixedMul, FixedDiv. BSP coordinates are stored as fixed-point; this is essential. |

## BSP Traversal & Rendering

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/r_bsp.c` | **Core BSP traversal engine** | R_RenderBSPNode() is the main BSP tree traversal function. R_Subsector() processes BSP leaves. R_CheckBBox() performs frustum/visibility culling. R_AddLine() clips segments to screen. This is the most critical BSP file. |
| `linuxdoom-1.10/r_bsp.h` | Declares BSP traversal functions | Exports R_RenderBSPNode, R_ClearClipSegs, R_ClearDrawSegs. Public interface to BSP rendering. |
| `linuxdoom-1.10/r_segs.c` | Segment drawing and clipping | R_StoreWallRange() records walls to be drawn; implements clipping logic. R_ClipSolidWallSegment() and R_ClipPassWallSegment() manage screen coverage. Directly called by R_AddLine(). |
| `linuxdoom-1.10/r_segs.h` | Declares segment rendering functions | Exports drawing and clipping routines. |
| `linuxdoom-1.10/r_plane.c` / `r_plane.h` | Floor/ceiling rendering | R_FindPlane() caches planes per subsector. Planes are derived from subsector sectors (BSP leaves determine which sector's floor/ceiling is visible). |

## Point-on-Side / Spatial Testing

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/r_main.c` | **BSP utility functions** | R_PointOnSide() tests which side of a partition line a point lies on (used in BSP traversal and clipping). Also initializes view parameters (viewx, viewy, viewz, viewangle) needed for BSP tree traversal. |
| `linuxdoom-1.10/r_main.h` | Declares view and BSP utilities | Exports R_PointOnSide and rendering setup functions. |
| `linuxdoom-1.10/m_bbox.h` | Bounding box utilities | M_ClearBox(), M_AddToBox(); used for bbox operations. Bounding boxes are stored in BSP nodes and used for culling. |

## Collision & Spatial Queries

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/p_map.c` | Collision detection and movement | Uses blockmap (spatial hash) for movement/collision checks. Also performs segment-level tests and sight checks. Blockmap is complementary to BSP for spatial queries. |
| `linuxdoom-1.10/p_maputl.c` | Map utility functions; blockmap operations | BlockmapSearch, spatial iteration. Supports collision queries. |
| `linuxdoom-1.10/p_local.h` | Play module declarations | Defines MAP_* constants (MAPBLOCKUNITS, MAPBLOCKSIZE, MAPBLOCKSHIFT). Needed for understanding blockmap layout. |

## Sight/Visibility

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/p_sight.c` | Line-of-sight testing | Performs detailed sight checks between map points. Uses segments and sectors; may leverage BSP structure for spatial optimization (though not directly obvious). |

## Game State & Loading

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/doomstat.c` / `doomstat.h` | Global state and map data pointers | Declares extern pointers: vertexes, segs, nodes, subsectors, sectors, lines, sides, blockmap, rejectmatrix. These are the global arrays populated by P_Setup* functions. |
| `linuxdoom-1.10/g_game.c` | Game loop; calls level setup and rendering | High-level orchestration; calls P_SetupLevel() and R_RenderPlayerView(). Shows how BSP is integrated into game flow. |

## Rendering Pipeline

| File | BSP Role | Why It Matters |
|------|----------|----------------|
| `linuxdoom-1.10/r_draw.c` / `r_draw.h` | Low-level pixel drawing | Called by segment/plane rendering. Implements column/span drawing. Not BSP-specific but executed as a result of BSP traversal. |
| `linuxdoom-1.10/r_things.c` | Sprite rendering | R_AddSprites() called per subsector (BSP leaf). Sprites are drawn after walls from that subsector are registered. |

## Notes

- **`p_setup.c`** and **`r_bsp.c`** are the two most critical files for understanding BSP. 
- **`r_defs.h`** and **`doomdata.h`** must be read in parallel to understand data layout.
- **`r_main.c`** is essential for understanding point-on-side testing and view setup, which drive BSP traversal.
- **`p_map.c`** shows how BSP interacts with collision and gameplay but is secondary to rendering.
