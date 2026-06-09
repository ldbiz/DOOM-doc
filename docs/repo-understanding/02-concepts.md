# 02 - BSP Concepts & Glossary

## Core BSP concepts

### BSP node

- **Meaning in DOOM**: An internal node in the binary space partition tree. Each node splits 2D space with a partition line; child subtrees represent the front and back half-spaces.
- **In this repo**: Represented by `node_t` (runtime, `r_defs.h`) and `mapnode_t` (on-disk, `doomdata.h`). Fields: partition line origin `(x,y)` and delta `(dx,dy)`, two child bounding boxes `bbox[2][4]`, and two child indices `children[2]`.
- **Where**: Defined in `r_defs.h` (runtime) and `doomdata.h` (WAD). Loaded by `P_LoadNodes()` in `p_setup.c`. Traversed by `R_RenderBSPNode()` in `r_bsp.c` and `P_CrossBSPNode()` in `p_sight.c`.
- **Related**: `subsector_t` (leaf nodes), `NF_SUBSECTOR` (leaf marker flag)

### Child node / subsector marker (`NF_SUBSECTOR`)

- **Meaning in DOOM**: A compact encoding where `children[]` entries are `unsigned short` values. If the high bit (0x8000) is set, the value is a subsector index (leaf). Otherwise, it's an index into the `nodes[]` array.
- **In this repo**: `#define NF_SUBSECTOR 0x8000` in `doomdata.h`. Checked in `r_bsp.c`: `if (bspnum & NF_SUBSECTOR)` then `R_Subsector(bspnum & ~NF_SUBSECTOR)`. Used identically in `p_sight.c`, `r_main.c`.
- **Where**: `doomdata.h` (definition), `r_bsp.c` (render traversal), `p_sight.c` (LOS check), `r_main.c` (`R_PointInSubsector`)
- **Related**: `node_t.children[]`, `subsector_t`

### Subsector

- **Meaning in DOOM**: A convex polygon formed by the leaf regions of the BSP tree. Each subsector contains a list of segs that form its boundary. Subsectors are the finest granularity of the BSP — traversal stops at subsectors.
- **In this repo**: `subsector_t` (`r_defs.h`): `sector_t* sector`, `short numlines`, `short firstline`. The `firstline` index points into the `segs[]` array; `numlines` segs belong to this subsector. The loader (`P_LoadSubsectors()` in `p_setup.c`) populates from `mapsubsector_t`.
- **Where**: `r_defs.h`, `p_setup.c` (loading), `r_bsp.c` (`R_Subsector()` processes), `p_sight.c` (`P_CrossSubsector()` checks)
- **Related**: `seg_t`, `sector_t`, `node_t` (BSP nodes partition space; subsectors are leaves)

### Seg

- **Meaning in DOOM**: A "segment" — a portion of a linedef that has been split by the BSP builder so that each seg lies entirely within a single subsector. Segs are the atomic wall units that the renderer processes.
- **In this repo**: `seg_t` (`r_defs.h`): `vertex_t *v1, *v2` (endpoints), `fixed_t offset` (distance along linedef), `angle_t angle`, `side_t *sidedef`, `line_t *linedef`, `sector_t *frontsector, *backsector`. On-disk: `mapseg_t` (`doomdata.h`) with 16-bit indices.
- **Where**: `r_defs.h` (runtime struct), `doomdata.h` (WAD struct), `p_setup.c` (`P_LoadSegs()`), `r_bsp.c` (processed in `R_AddLine()`), `r_segs.c` (clipped into drawsegs)
- **Related**: `drawseg_t` (per-frame rendered representation), `subsector_t` (segs are grouped into subsectors), `linedef_t` (segs are fragments of linedefs)

### Linedef

- **Meaning in DOOM**: A line definition from the map editor — a wall, trigger, or boundary between two vertices. Linedefs are split into segs by the BSP builder; the game engine works primarily with segs at runtime for rendering, but uses linedefs for collision and game logic.
- **In this repo**: `line_t` (`r_defs.h`): `vertex_t *v1, *v2`, precomputed `dx, dy`, `short flags/special/tag`, `short sidenum[2]` (indices into sides array; `-1` if one-sided), `fixed_t bbox[4]`, `slopetype_t slopetype`, `sector_t *frontsector, *backsector`. On-disk: `maplinedef_t` (`doomdata.h`).
- **Where**: `r_defs.h`, `doomdata.h`, `p_setup.c` (`P_LoadLineDefs()`, `P_GroupLines()`)
- **Related**: `seg_t` (segs are BSP-split linedef fragments), `side_t`, `sector_t`

### Sidedef

- **Meaning in DOOM**: Defines the texture and appearance of one side of a linedef. A one-sided line has one sidedef; a two-sided line has two (front and back).
- **In this repo**: `side_t` (`r_defs.h`): `fixed_t textureoffset, rowoffset`, texture indices (`toptexture`, `bottomtexture`, `midtexture`), `sector_t *sector`. On-disk: `mapsidedef_t` (`doomdata.h`).
- **Where**: `r_defs.h`, `doomdata.h`, `p_setup.c` (`P_LoadSideDefs()`)
- **Related**: `line_t` (linedefs reference sidedefs via `sidenum[]`), `seg_t` (segs carry a `sidedef` pointer)

### Sector

- **Meaning in DOOM**: A closed area with uniform floor/ceiling height, flat texture, and light level. Sectors are the basic spatial units of the game world — each subsector belongs to exactly one sector.
- **In this repo**: `sector_t` (`r_defs.h`): `fixed_t floorheight, ceilingheight`, `short floorpic, ceilingpic, lightlevel, special, tag`, `mobj_t *thinglist` (linked list), `line_t **lines` (array of bounding lines). On-disk: `mapsector_t` (`doomdata.h`).
- **Where**: `r_defs.h`, `doomdata.h`, `p_setup.c` (`P_LoadSectors()`, `P_GroupLines()`)
- **Related**: `subsector_t` (each subsector points to one sector), `visplane_t` (floor/ceiling spans grouped by sector properties), `seg_t` (segs carry front/back sector pointers)

### Vertex

- **Meaning in DOOM**: A 2D coordinate point. Lines, segs, and BSP node partition lines all reference vertices.
- **In this repo**: `vertex_t` (`r_defs.h`): `fixed_t x, y`. On-disk: `mapvertex_t` (`doomdata.h`): `short x, y`.
- **Where**: `r_defs.h`, `doomdata.h`, `p_setup.c` (`P_LoadVertexes()` converts shorts to fixed-point via `<<FRACBITS`)
- **Related**: `line_t`, `seg_t`, `node_t` (partition line defined by origin + delta)

### Partition line

- **Meaning in DOOM**: Each BSP node divides space along an infinite line. The partition line is defined by an origin point `(x,y)` and a direction vector `(dx,dy)`. One side is "front" (child 0), the other is "back" (child 1).
- **In this repo**: Stored as fields in `node_t`: `x, y, dx, dy` (all `fixed_t`). The side test is `R_PointOnSide()` in `r_main.c`: computes cross product sign of `(point - origin)` with `(dx,dy)`.
- **Where**: `r_defs.h` (struct fields), `r_main.c` (`R_PointOnSide()`), `p_sight.c` (`P_DivlineSide()`)
- **Related**: `node_t`, `NF_SUBSECTOR` (child indices encode front/back subtrees)

### Bounding box

- **Meaning in DOOM**: Each BSP node stores axis-aligned bounding boxes for its two children, used for frustum culling during traversal. If a child's bbox is off-screen, the renderer skips that subtree entirely.
- **In this repo**: `node_t.bbox[2][4]` — two bboxes, each 4 `fixed_t` values (left, top, right, bottom). Checked by `R_CheckBBox()` in `r_bsp.c` against the current view frustum. On-disk: `mapnode_t.bbox[2][4]` (shorts, scaled up during loading).
- **Where**: `r_defs.h` (field), `r_bsp.c` (`R_CheckBBox()`), `p_setup.c` (loaded and scaled `<<FRACBITS`)
- **Related**: `node_t`, `m_bbox.c` (bounding box utilities)

### Map lump

- **Meaning in DOOM**: A named data chunk within a WAD file. Each map's BSP data is stored across three specific lumps: `NODES`, `SSECTORS` (subsectors), and `SEGS`.
- **In this repo**: Lump names are referenced by index via the `ML_*` enum in `doomdata.h`: `ML_NODES`, `ML_SSECTORS`, `ML_SEGS`, along with `ML_VERTEXES`, `ML_LINEDEFS`, `ML_SIDEDEFS`, `ML_SECTORS`, `ML_REJECT`, `ML_BLOCKMAP`. Loaded via `W_CacheLumpNum(lumpnum + ML_*)` in `P_SetupLevel()`.
- **Where**: `doomdata.h` (enum), `p_setup.c` (loading), `w_wad.c` (caching)
- **Related**: All on-disk map types

---

## Render-time BSP concepts

### Drawseg

- **Meaning in DOOM**: A per-frame record of a visible wall segment, created during BSP traversal. Each drawseg stores the screen X range, per-pixel scale values, silhouette flags, and pointers to sprite-clipping data.
- **In this repo**: `drawseg_t` (`r_defs.h`). Pooled in `drawsegs[MAXDRAWSEGS]` array (`r_bsp.c`). Created by `R_StoreWallRange()` in `r_segs.c` during seg processing. Later consumed by `R_DrawSprite()` for per-column sprite occlusion clipping.
- **Where**: `r_defs.h` (struct), `r_bsp.c` (array, `ds_p` pointer), `r_segs.c` (creation), `r_things.c` (consumption for sprite clipping)
- **Related**: `seg_t` (the map-level segment), `visplane_t`, `vissprite_t`

### Visplane

- **Meaning in DOOM**: A horizontal span accumulator for floors and ceilings. During BSP traversal, the renderer groups visible floor/ceiling fragments by matching (height, flat texture, light level) into visplanes. After traversal, all visplanes are drawn in batches as horizontal spans.
- **In this repo**: `visplane_t` (`r_defs.h`): `fixed_t height`, `int picnum, lightlevel, minx, maxx`, `byte top[SCREENWIDTH], bottom[SCREENWIDTH]`. Pooled in `visplanes[MAXVISPLANES]` (`r_plane.c`). Created/merged by `R_FindPlane()` and `R_CheckPlane()`. Drawn by `R_DrawPlanes()` → `R_MakeSpans()` → `R_MapPlane()`.
- **Where**: `r_defs.h` (struct), `r_plane.c` (management and drawing)
- **Related**: `sector_t` (visplane height/texture come from sector), `R_Subsector()` (sets up visplanes per subsector)

### Vissprite

- **Meaning in DOOM**: A projected sprite (map thing) record, created during BSP subsector processing. Sprites are collected per-subsector during traversal, then sorted by distance and drawn back-to-front after visplanes.
- **In this repo**: `vissprite_t` (`r_defs.h`): screen range `x1/x2`, world position `gx/gy/gz`, `scale`, `texturemid`, `patch` index, `colormap`. Pooled in `vissprites[MAXVISSPRITES]` (`r_things.c`). Created by `R_ProjectSprite()` (called from `R_AddSprites()` inside `R_Subsector()`). Drawn by `R_DrawSprite()`.
- **Where**: `r_defs.h` (struct), `r_things.c` (creation, sorting, drawing)
- **Related**: `mobj_t` (the map object being projected), `drawseg_t` (used for per-column sprite clipping)

---

## Spatial-query BSP concepts

### REJECT table

- **Meaning in DOOM**: A precomputed bitmask table stored in the WAD that maps sector-pair visibility. If the bit for sector pair (A,B) is set, line-of-sight between those sectors is known-impossible without further checking. Allows `P_CheckSight()` to short-circuit expensive BSP traversal.
- **In this repo**: Loaded as raw byte array `rejectmatrix` in `P_SetupLevel()` (`p_setup.c`): `rejectmatrix = W_CacheLumpNum(lumpnum + ML_REJECT, PU_LEVEL)`. Checked in `P_CheckSight()` (`p_sight.c`) before falling through to BSP traversal. No `P_LoadReject()` function — loaded directly.
- **Where**: `p_setup.c` (loading), `p_sight.c` (checking)
- **Related**: `P_CheckSight()`, `P_CrossBSPNode()`

### BLOCKMAP

- **Meaning in DOOM**: A 2D grid-based spatial index for collision detection. The map is divided into fixed-size blocks; each block lists the lines and things that intersect it. Used for movement collision, not BSP traversal. Included here for contrast: BSP = rendering/LOS, BLOCKMAP = collision.
- **In this repo**: Loaded by `P_LoadBlockMap()` in `p_setup.c`: `blockmaplump`, `blockmap`, `blocklinks[]`, `bmaporgx/y`, `bmapwidth/height`. Queried by `P_BlockLinesIterator()` and `P_BlockThingsIterator()` in `p_maputl.c`.
- **Where**: `p_setup.c`, `p_maputl.c`
- **Related**: Not BSP. Included for clarity on spatial-query responsibility separation.

---

## Key constants

| Constant | Value | File | Purpose |
|---|---|---|---|
| `NF_SUBSECTOR` | `0x8000` | `doomdata.h` | High bit on `children[]` entries marks subsector leaf nodes |
| `FRACBITS` | `16` | `m_fixed.h` | Bits of fractional precision in `fixed_t` |
| `FRACUNIT` | `1 << 16` | `m_fixed.h` | The value `1.0` in fixed-point |
| `MAXDRAWSEGS` | `256` | `r_defs.h` | Max wall drawsegs per frame |
| `MAXVISPLANES` | `128` | `r_plane.c` | Max floor/ceiling visplanes per frame |
| `MAXVISSPRITES` | `128` | `r_things.h` | Max projected sprites per frame |
| `MAXSEGS` | `32` | `r_bsp.c` | Max solid clip ranges during wall clipping |
| `MAXOPENINGS` | `SCREENWIDTH*64` | `r_plane.c` | Size of `openings[]` buffer for sprite-clip data |
| `SCREENWIDTH` | `320` | `doomdef.h` | Screen width in pixels; sizes visplane top/bottom and clip arrays |

## Relationship summary

```
BSP Tree (nodes[] + subsectors[] + segs[])
    │
    ├── Rendering: R_RenderBSPNode() → R_Subsector()
    │       ├── visplane_t (floor/ceiling spans)
    │       ├── drawseg_t  (wall segments with clip data)
    │       └── vissprite_t (projected sprites)
    │
    └── Visibility: P_CheckSight()
            ├── REJECT table (coarse filter)
            └── P_CrossBSPNode() → P_CrossSubsector() (precise seg test)

BLOCKMAP (separate: blockmap[] + blocklinks[][])
    └── Collision/Movement: P_CheckPosition() → P_BlockLinesIterator()
```
