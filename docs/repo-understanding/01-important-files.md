# Important BSP-related files

## Map/WAD/lump loading

| File / directory | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/doomdata.h` | Defines map-lump order and packed records for nodes, subsectors, segs, vertices, linedefs, sidedefs, sectors, `REJECT`, and `BLOCKMAP`. | This is the contract expected from a Doom-format map and defines `NF_SUBSECTOR`. |
| `linuxdoom-1.10/w_wad.c` | Builds the lump directory, resolves names, reads/caches lumps, and supports reloadable development WADs. | BSP loaders receive lump numbers and bytes through this layer; later WADs override earlier same-named lumps. |
| `linuxdoom-1.10/w_wad.h` | Declares WAD directory/cache operations and lump metadata. | Useful when changing how BSP-related lump data enters `p_setup.c`. |
| `linuxdoom-1.10/p_setup.c` | Loads every BSP-adjacent map lump, converts records to runtime types, cross-links segs/lines/sides/sectors, and groups subsectors. | The central load-time integration point for the BSP subsystem. |
| `linuxdoom-1.10/g_game.c` | `G_DoLoadLevel` triggers `P_SetupLevel` for the selected episode/map. | Establishes when BSP arrays are replaced and become available. |
| `linuxdoom-1.10/d_main.c` | Selects IWAD/PWAD files and map-start arguments. | Controls which WAD and map-marker data ultimately supplies the BSP. |

## BSP data structures

| File / directory | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/r_defs.h` | Defines runtime `node_t`, `subsector_t`, `seg_t`, linked map geometry, `drawseg_t`, and `visplane_t`. | Shows how packed lump indexes become fixed-point values and direct pointers used at runtime. |
| `linuxdoom-1.10/p_local.h` | Declares play-side map arrays, REJECT/BLOCKMAP state, and map-block scale constants. | Defines the shared play-side interface around BSP-adjacent spatial data. |
| `linuxdoom-1.10/r_state.h` | Exposes loaded map arrays to renderer modules. | Connects setup-owned arrays to traversal and drawing code. |

## BSP traversal

| File / directory | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/r_bsp.c` | Recursively renders the BSP front-to-back, tests child bounding boxes, processes subsectors, and clips visible segs. | The primary visual BSP traversal implementation. |
| `linuxdoom-1.10/r_main.c` | Implements partition side tests, `R_PointInSubsector`, frame setup, and the `R_RenderBSPNode` root call. | Contains both the general point-location descent and per-frame traversal entry. |
| `linuxdoom-1.10/p_sight.c` | Traverses BSP nodes/subsectors along a sight trace after checking `REJECT`. | The principal non-rendering full BSP traversal. |

## Rendering and visibility products

| File / directory | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/r_segs.c` | Converts visible seg screen ranges into wall drawing, drawsegs, sprite silhouettes, and plane bounds. | Explains what BSP-discovered wall segments become. |
| `linuxdoom-1.10/r_plane.c` | Maintains and draws visplanes accumulated while rendering subsectors/walls. | Explains how subsector sector floors and ceilings become rendered spans. |
| `linuxdoom-1.10/r_things.c` | Adds sector things during subsector visits and later clips sprites against drawsegs. | Shows why BSP rendering order and drawseg output matter to sprites. |

## Collision and spatial queries

| File / directory | BSP-related role | Why it matters |
|---|---|---|
| `linuxdoom-1.10/p_maputl.c` | Uses `R_PointInSubsector` to link things to sectors, but uses BLOCKMAP iteration for line/thing path queries. | Defines the practical boundary between BSP use and collision-grid use. |
| `linuxdoom-1.10/p_map.c` | Uses point-to-subsector results for movement floor/ceiling context and BLOCKMAP for collision candidates. | Prevents incorrectly treating all collision as BSP-driven. |
| `linuxdoom-1.10/p_mobj.c` | Uses `R_PointInSubsector` when spawning/removing objects and selecting floor heights. | Demonstrates gameplay dependence on BSP point location. |
