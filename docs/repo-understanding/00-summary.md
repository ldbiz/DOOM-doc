# BSP Partitioning: Summary

## Overview

This DOOM source codebase uses Binary Space Partitioning (BSP) as the core spatial organization and rendering system. BSP is employed for two primary purposes:

1. **Rendering**: BSP trees guide front-to-back traversal of the map to determine which walls, floors, ceilings, and sprites are visible from the player's viewpoint.
2. **Spatial Queries**: BSP data structures and related spatial acceleration structures (blockmap, bounding boxes) support gameplay mechanics including collision detection and line-of-sight queries.

The BSP system is deeply embedded across the codebase—BSP nodes and subsectors are the primary organizational unit for map geometry, and virtually all rendering and spatial queries depend on BSP relationships.

## Role in the Architecture

**Rendering Pipeline**: 
The renderer starts at the BSP tree root and recursively traverses nodes to find leaf subsectors. At each subsector, the engine collects visible walls (line segments), floors, ceilings, and sprites, then draws them in back-to-front order to ensure correct depth. The BSP traversal also manages screen space clipping to avoid overdrawing.

**Map Loading**:
When a map loads, all BSP data (nodes, subsectors, segments, vertexes, linedefs, sidedefs, sectors) is deserialized from WAD lumps and converted from fixed-point 16-bit format to the runtime 32-bit fixed-point representation for precision.

**Gameplay Support**:
Collision detection uses BSP nodes in conjunction with the blockmap (a 2D grid overlay) to test whether moving objects intersect walls or each other. Line-of-sight queries and spatial bounding box checks also leverage BSP structure.

## Main Data Structures

### On-Disk Format (WAD Lumps)
- **`mapvertex_t`**: Vertex coordinates (16-bit x, y)
- **`maplinedef_t`**: Linedef with vertex indices, flags, special, and sidedef indices
- **`mapsidedef_t`**: Sidedef with textures, offsets, and sector index
- **`mapseg_t`**: Line segment (fragment of a linedef), with vertices, angle, linedef reference, and side
- **`mapsubsector_t`**: Leaf node of BSP; contains count and start index of segs
- **`mapnode_t`**: Interior BSP node with partition line (x, y, dx, dy), two child references, and two bounding boxes
- **`mapsector_t`**: Sector definition with floor/ceiling heights, textures, light level, special, tag

### Runtime Format (In-Memory)
- **`vertex_t`**: Runtime vertex in 32-bit fixed-point (fixed_t x, y)
- **`line_t`**: Linedef with pointers to vertices and sidedefs, derived dx/dy, bounding box, slope type, and sector references
- **`side_t`**: Sidedef with texture indices (not names), offsets, and sector pointer
- **`seg_t`**: Segment with vertex pointers, linedef/sidedef pointers, both sectors
- **`subsector_t`**: Subsector with sector pointer and firstline/numlines
- **`node_t`**: BSP node in 32-bit fixed-point, with children array marked by NF_SUBSECTOR flag
- **`sector_t`**: Sector with heights, light level, special, tag, sound target, and list of lines/things

### Supporting Structures
- **`drawseg_t`**: Screen-space record of a wall segment being rendered (position, scale, clipping info)
- **`visplane_t`**: Record of a floor or ceiling plane at a given height and light level (used for rendering)

## Main Files and Modules

| Purpose | Core Files |
|---------|-----------|
| **BSP Traversal & Rendering** | `r_bsp.c/h` (main traversal loop), `r_segs.c/h` (segment drawing) |
| **Map/WAD Loading** | `p_setup.c/h` (map loading), `w_wad.c/h` (WAD I/O) |
| **BSP Data Structures** | `r_defs.h` (runtime types), `doomdata.h` (WAD types) |
| **Rendering Setup** | `r_main.c/h` (viewpoint, angle calculations, BSP utilities) |
| **Collision/Spatial** | `p_map.c/h` (movement, collision), `p_maputl.c` (spatial utilities) |

## Main Runtime Flow

```
Game Start
  ↓
P_SetupLevel(level)
  ├─ P_LoadVertexes(lump)      → vertexes[] from VERTEXES lump
  ├─ P_LoadLineDefs(lump)       → lines[] from LINEDEFS lump
  ├─ P_LoadSideDefs(lump)       → sides[] from SIDEDEFS lump
  ├─ P_LoadSegs(lump)           → segs[] from SEGS lump
  ├─ P_LoadSubsectors(lump)     → subsectors[] from SSECTORS lump
  ├─ P_LoadNodes(lump)          → nodes[] from NODES lump (BSP tree)
  ├─ P_LoadSectors(lump)        → sectors[] from SECTORS lump
  ├─ P_LoadBlockMap(lump)       → blockmap for spatial queries
  └─ P_LoadReject(lump)         → rejectmatrix for sight checks

Per-Frame Rendering
  ↓
R_RenderPlayerView()
  ├─ R_SetupFrame(player)       → set viewx, viewy, viewz, viewangle
  ├─ R_ClearClipSegs()          → initialize screen clipping
  ├─ R_RenderBSPNode(root)      → recursively traverse BSP tree
  │   └─ For each subsector:
  │       ├─ R_Subsector()      → determine planes, add sprites
  │       └─ R_AddLine()        → clip and register segments
  ├─ R_DrawPlanes()             → draw floors/ceilings
  └─ R_DrawMaskedSegs()         → draw two-sided wall midtextures

Movement/Collision
  ↓
P_Move(thing, x, y)
  ├─ P_CheckPosition()          → test against blockmap/BSP
  └─ Updates thing position if valid
```

## BSP-Related Behavior

The BSP system supports the following key behaviors:

1. **Front-to-Back Visibility Ordering**: The renderer traverses the BSP tree recursively, starting at the viewpoint's node and recursing front-first, then back (with visibility checks). This ensures walls are drawn in back-to-front screen order without explicit z-sorting.

2. **Screen Clipping Management**: As segments are added, a clip list (solidsegs) tracks which screen columns have been filled. Pass-through segments clip against this list but don't block; solid segments update it. This avoids overdrawing sky or distant areas.

3. **Subsector Classification**: Each subsector references a single sector and a range of segments. All geometry in a subsector belongs to one sector, allowing simple flat (floor/ceiling) rendering.

4. **Spatial Subdivision**: The blockmap (separate from BSP) provides 2D spatial hashing for fast collision queries, partitioning the map into fixed-size blocks and storing thing/line references per block.

5. **Bounding Box Culling**: Each BSP node stores bounding boxes for both child subtrees, allowing quick rejection of entire subtrees if their bounding box is off-screen (via `R_CheckBBox`).

## BSP in Rendering vs. Collision

- **Rendering**: Uses BSP tree for traversal order and visibility determination. Every frame, the tree is traversed from the root, starting from the player's current node.
- **Collision**: Primarily uses blockmap for spatial hashing and linear searches through segments and things. BSP is less directly involved, though bounding boxes and sector knowledge inform spatial queries.

## Read These First

Understanding BSP in this codebase requires reading the following in order:

1. **`linuxdoom-1.10/doomdata.h`** — WAD file format definitions (map lumps and on-disk structures)
2. **`linuxdoom-1.10/r_defs.h`** — Runtime data structures (node_t, seg_t, subsector_t, etc.)
3. **`linuxdoom-1.10/p_setup.c`** — Map loading functions (P_LoadNodes, P_LoadSegs, etc.)
4. **`linuxdoom-1.10/r_bsp.c`** — BSP traversal, the R_RenderBSPNode function, and R_Subsector
5. **`linuxdoom-1.10/r_main.c`** — Point-on-side testing (R_PointOnSide) and view setup
6. **`linuxdoom-1.10/r_segs.c`** — Segment rendering and clipping logic
7. **`linuxdoom-1.10/p_map.c`** — Collision detection and spatial queries
8. **`linuxdoom-1.10/m_bbox.h`** — Bounding box utilities

## Notes

- This is the *original id Software DOOM 1.10 source code* under GPLv2. All BSP implementation details reflect the original engine design from 1993–1996.
- The codebase uses 16-bit fixed-point for map coordinates and 32-bit fixed-point for runtime calculations.
- No BSP builder is included in this repository; BSP trees are pre-computed by external tools (e.g., WadAuthor, ZenNode) and embedded in the WAD file.
