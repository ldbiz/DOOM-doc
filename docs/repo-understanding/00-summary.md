# BSP subsystem summary

## Role of BSP partitioning

This repository consumes a prebuilt Doom-format BSP; it does not build or edit one. A map's `NODES`, `SSECTORS`, and `SEGS` lumps encode the tree and its leaves. `P_SetupLevel` decodes those records into pointer-rich runtime structures linked to vertices, linedefs, sidedefs, and sectors.

The BSP has three direct runtime roles:

- **Front-to-back rendering:** each frame starts at the root node, visits the viewer's side first, rejects fully occluded back-side bounding boxes, and turns reached subsectors into walls, planes, and sprites.
- **Point-to-region lookup:** gameplay code descends the tree to find the subsector and sector containing an `(x, y)` position. This places things in sector lists and obtains floor/ceiling information.
- **Line of sight:** `P_CheckSight` first uses `REJECT`, then traverses BSP nodes and subsector segs to determine whether walls or vertical openings block sight.

General movement collision and attack/use traces are **not BSP traversals**. They primarily use the separate `BLOCKMAP` grid and linedefs. The BSP still supports gameplay indirectly by locating a point's subsector/sector.

## Main BSP-related data structures

| Layer | Structures | Purpose |
|---|---|---|
| On-disk map format | `mapnode_t`, `mapsubsector_t`, `mapseg_t`, `mapvertex_t`, `maplinedef_t`, `mapsidedef_t`, `mapsector_t` | Packed, short-based WAD lump records and map-lump ordering. |
| Runtime map geometry | `node_t`, `subsector_t`, `seg_t`, `vertex_t`, `line_t`, `side_t`, `sector_t` | Fixed-point geometry and direct pointers used by renderer and play code. |
| Renderer products | `drawseg_t`, `visplane_t`, solid clip ranges | Screen-space wall/sprite clipping and floor/ceiling accumulation produced while visiting subsectors. |
| Adjacent spatial data | `rejectmatrix`, `blockmaplump`, `blockmap`, `blocklinks` | Fast sector-pair sight rejection and grid-based collision/path traversal. |

A `node_t` stores a partition line, two child bounding boxes, and two child identifiers. A child whose `NF_SUBSECTOR` bit is set names a leaf. A `subsector_t` names a consecutive run of `seg_t` records and is assigned a sector after loading.

## Main modules

- `linuxdoom-1.10/doomdata.h` — persistent map-lump layout, lump order, and the subsector child marker.
- `linuxdoom-1.10/r_defs.h` — runtime BSP/map/rendering structures.
- `linuxdoom-1.10/p_setup.c` — level-lump decoding, cross-linking, and lifetime of BSP arrays.
- `linuxdoom-1.10/r_main.c` — point-to-subsector descent and per-frame renderer entry point.
- `linuxdoom-1.10/r_bsp.c` — front-to-back tree traversal, subtree bounding-box visibility checks, and subsector submission.
- `linuxdoom-1.10/r_segs.c`, `r_plane.c`, and `r_things.c` — consume visible subsector content as walls, planes, and sprites.
- `linuxdoom-1.10/p_sight.c` — `REJECT` plus BSP traversal for line of sight.
- `linuxdoom-1.10/p_maputl.c` and `p_map.c` — show the boundary between BSP point location and BLOCKMAP-based collision/path queries.

## Runtime flow

1. Startup selects IWAD/PWAD files and builds a global lump directory through `w_wad.c`.
2. A level transition calls `G_DoLoadLevel`, then `P_SetupLevel`.
3. `P_SetupLevel` finds the `E#M#` or `MAP##` marker and treats fixed offsets after it as map lumps.
4. Loader functions convert little-endian, 16-bit map records into level-lifetime fixed-point arrays and pointer relationships.
5. `P_GroupLines` assigns each subsector's sector from its first seg and builds sector line lists/block bounds.
6. Each rendered frame calls `R_RenderBSPNode(numnodes - 1)`, where the last node is assumed to be the root.
7. Gameplay uses the same arrays for `R_PointInSubsector` and `P_CheckSight`; most collision/path traversal uses `BLOCKMAP` instead.

## Where BSP is used

| Concern | BSP involvement |
|---|---|
| Map loading | Direct: loads `SEGS`, `SSECTORS`, and `NODES`, then links them to other map structures. |
| Rendering | Direct and central: controls subsector visitation order and prunes occluded subtrees. |
| Visibility | Direct for line of sight; adjacent `REJECT` matrix provides an earlier sector-level rejection. |
| Collision/movement | Limited: point-to-subsector lookup supplies sector heights; broad spatial line/thing iteration uses `BLOCKMAP`. |
| BSP construction | None found; nodes, subsectors, and segs must already exist in WAD data. |

## Read these first

1. `linuxdoom-1.10/doomdata.h`
2. `linuxdoom-1.10/r_defs.h`
3. `linuxdoom-1.10/p_setup.c`
4. `linuxdoom-1.10/r_bsp.c`
5. `linuxdoom-1.10/r_main.c`
6. `linuxdoom-1.10/r_segs.c`
7. `linuxdoom-1.10/p_sight.c`
8. `linuxdoom-1.10/p_maputl.c`
9. `linuxdoom-1.10/w_wad.c`
