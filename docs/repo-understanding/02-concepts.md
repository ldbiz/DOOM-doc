# BSP & Map Concepts Glossary

## Core BSP Concepts

### BSP (Binary Space Partitioning)
**In Doom/General**:  
A tree structure that recursively partitions 3D (or 2D) space using axis-aligned planes. Each node contains a partition line; space is divided into front and back children. Leaves contain convex regions.

**In This Repo**:  
The BSP tree is the primary data structure for map organization and rendering. Stored as an array of `node_t` structures, accessed via indices. The tree is traversed from the root during every frame render. The root index is hardcoded or determined by WAD metadata; traversal uses recursive `R_RenderBSPNode()`. The tree partitions the map into convex convex subsectors (BSP leaves), each referencing a single sector and a list of wall segments.

**Related Files**:  
`r_bsp.c` (traversal), `p_setup.c` (loading), `r_defs.h` (structure definition)

**Related Concepts**:  
Node, subsector, partition line, children array, bounding box

---

### Node (BSP Node) / Interior Node
**In Doom/General**:  
An interior node of the BSP tree; contains a partition line and references to two child nodes (which may themselves be nodes or subsector leaves).

**In This Repo**:  
Represented as `node_t`:
```c
typedef struct {
    fixed_t x, y, dx, dy;              // Partition line: (x, y) to (x+dx, y+dy)
    fixed_t bbox[2][4];                // Bounding boxes for each child
    unsigned short children[2];        // Indices to children (nodes or subsectors)
} node_t;
```

During rendering, `R_RenderBSPNode(int bspnum)` tests whether the viewpoint is on the front or back side of the partition and recursively visits children. The `NF_SUBSECTOR` flag in `bspnum` indicates if a child is a subsector leaf.

**Related Files**:  
`r_defs.h`, `p_setup.c` (P_LoadNodes), `r_bsp.c` (R_RenderBSPNode)

**Related Concepts**:  
Partition line, front/back child, subsector, NF_SUBSECTOR flag

---

### Subsector (BSP Leaf / Leaf Node)
**In Doom/General**:  
A leaf (terminal node) of the BSP tree; represents a convex region of space. In Doom, all geometry within a subsector belongs to one sector and is rendered as a contiguous group of wall segments.

**In This Repo**:  
Represented as `subsector_t`:
```c
typedef struct subsector_s {
    sector_t*  sector;       // The sector containing this subsector
    short      numlines;     // Number of segments
    short      firstline;    // Index into segs[] array
} subsector_t;
```

When BSP traversal reaches a subsector (indicated by `NF_SUBSECTOR` flag), `R_Subsector(int num)` is called to:
- Look up the sector's floor/ceiling planes
- Add sprites in that sector
- Process all wall segments in that subsector

**Related Files**:  
`r_defs.h`, `p_setup.c` (P_LoadSubsectors), `r_bsp.c` (R_Subsector)

**Related Concepts**:  
BSP node, sector, segment, NF_SUBSECTOR flag

---

### Partition Line
**In Doom/General**:  
The infinite line used to partition space at a BSP node. Everything on one side goes to one child; everything on the other goes to the other child.

**In This Repo**:  
Stored in `node_t` as a point (x, y) and direction vector (dx, dy):
```c
fixed_t x = node->x;
fixed_t y = node->y;
fixed_t dx = node->dx;
fixed_t dy = node->dy;
// Line extends from (x, y) to (x+dx, y+dy) and infinitely in both directions
```

Point-on-side testing is performed by `R_PointOnSide()` in `r_main.c` using the cross product: `(px-x)*dy - (py-y)*dx`. Result < 0 → side 0 (front), ≥ 0 → side 1 (back).

**Related Files**:  
`r_main.c` (R_PointOnSide), `r_defs.h`

**Related Concepts**:  
Front/back side, node

---

### Front / Back Side / Child
**In Doom/General**:  
Relative to the partition line: front side is where the normal vector points; back side is opposite. Each side has a corresponding child node.

**In This Repo**:  
BSP nodes have a `children[2]` array. `children[0]` is the front child, `children[1]` is the back child. Rendering visits front first (recursively), then checks back visibility via `R_CheckBBox()`.

Example from `r_bsp.c`:
```c
side = R_PointOnSide(viewx, viewy, bsp);    // 0 = front, 1 = back
R_RenderBSPNode(bsp->children[side]);       // Visit viewpoint's side first
if (R_CheckBBox(bsp->bbox[side^1]))         // Check visibility of other side
    R_RenderBSPNode(bsp->children[side^1]); // Visit if visible
```

**Related Files**:  
`r_bsp.c`, `r_main.c` (R_PointOnSide)

**Related Concepts**:  
Node, partition line, NF_SUBSECTOR flag

---

### NF_SUBSECTOR Flag
**In Doom/General**:  
A flag bit (0x8000) used to mark node indices that refer to subsector leaves rather than interior nodes.

**In This Repo**:  
Defined in `doomdata.h`:
```c
#define NF_SUBSECTOR 0x8000
```

When traversing the BSP tree, if `bspnum & NF_SUBSECTOR` is true, the child is a subsector leaf. The actual subsector index is obtained by masking out the flag: `bspnum & (~NF_SUBSECTOR)`.

Example from `r_bsp.c`:
```c
if (bspnum & NF_SUBSECTOR) {
    if (bspnum == -1) R_Subsector(0);
    else R_Subsector(bspnum & (~NF_SUBSECTOR));
    return;
}
```

**Related Files**:  
`doomdata.h`, `r_bsp.c` (R_RenderBSPNode)

**Related Concepts**:  
Subsector, node

---

## Segment & Linedef Concepts

### Segment (LineSeg / Seg)
**In Doom/General**:  
A fragment of a linedef, created when the BSP builder splits linedefs at partition lines. A subsector contains a list of contiguous segments, all referencing the same sector.

**In This Repo**:  
Represented as `seg_t`:
```c
typedef struct {
    vertex_t*  v1, v2;       // Start and end vertices
    fixed_t    offset;       // Offset into the linedef (for texture alignment)
    angle_t    angle;        // Direction angle
    side_t*    sidedef;      // Sidedef (texture/sector info)
    line_t*    linedef;      // Parent linedef
    sector_t*  frontsector;  // Sector in front
    sector_t*  backsector;   // Sector behind (NULL for one-sided walls)
} seg_t;
```

During rendering, `R_AddLine()` processes each segment: determines visibility, clips to screen, and registers via `R_ClipSolidWallSegment()` or `R_ClipPassWallSegment()`.

**Related Files**:  
`r_defs.h`, `p_setup.c` (P_LoadSegs), `r_bsp.c` (R_AddLine)

**Related Concepts**:  
Linedef, sidedef, subsector, vertex

---

### Linedef (Line Definition)
**In Doom/General**:  
A line segment in the map editor. Linedefs represent walls, doors, special triggers. Each linedef references two vertices and one or two sidedefs (for one-sided or two-sided walls).

**In This Repo**:  
On-disk format (`maplinedef_t` in `doomdata.h`):
```c
typedef struct {
    short v1, v2;              // Vertex indices
    short flags;               // ML_BLOCKING, ML_TWOSIDED, ML_SECRET, etc.
    short special;             // Trigger special type
    short tag;                 // Sector tag for triggers
    short sidenum[2];          // Sidedef indices (sidenum[1]=-1 if one-sided)
} maplinedef_t;
```

Runtime format (`line_t` in `r_defs.h`):
```c
typedef struct line_s {
    vertex_t* v1, v2;         // Vertex pointers
    fixed_t dx, dy;            // Precalculated direction
    short flags, special, tag;
    short sidenum[2];
    fixed_t bbox[4];           // Bounding box
    slopetype_t slopetype;     // ST_HORIZONTAL, ST_VERTICAL, ST_POSITIVE, ST_NEGATIVE
    sector_t* frontsector, backsector;
    // ... other fields
} line_t;
```

Linedefs are loaded before segments (P_LoadLineDefs before P_LoadSegs). Segments reference linedefs to retrieve texture and sector info.

**Related Files**:  
`doomdata.h`, `r_defs.h`, `p_setup.c` (P_LoadLineDefs), `r_segs.c`

**Related Concepts**:  
Segment, sidedef, vertex, sector

---

### Sidedef (Side Definition)
**In Doom/General**:  
The visual appearance of one side of a linedef: textures for upper/middle/lower parts, offsets, and sector reference.

**In This Repo**:  
On-disk format (`mapsidedef_t` in `doomdata.h`):
```c
typedef struct {
    short textureoffset;       // Horizontal texture offset
    short rowoffset;           // Vertical texture offset
    char toptexture[8];        // Upper texture name
    char bottomtexture[8];     // Lower texture name
    char midtexture[8];        // Middle texture name
    short sector;              // Sector index this sidedef faces
} mapsidedef_t;
```

Runtime format (`side_t` in `r_defs.h`):
```c
typedef struct {
    fixed_t textureoffset, rowoffset;
    short toptexture, bottomtexture, midtexture;  // Texture indices (not names)
    sector_t* sector;
} side_t;
```

Each segment references a sidedef to determine which textures to draw. A linedef with two sidedefs is two-sided (wall visible from both sides); one sidedef makes it one-sided (opaque wall).

**Related Files**:  
`doomdata.h`, `r_defs.h`, `p_setup.c` (P_LoadSideDefs), `r_segs.c`

**Related Concepts**:  
Linedef, segment, sector, texture

---

## Sector & Vertex Concepts

### Sector
**In Doom/General**:  
A flat region of space with uniform floor/ceiling heights, light level, and floor/ceiling textures. Sectors define areas of the map; walls (linedefs) separate sectors.

**In This Repo**:  
On-disk format (`mapsector_t` in `doomdata.h`):
```c
typedef struct {
    short floorheight, ceilingheight;
    char floorpic[8], ceilingpic[8];
    short lightlevel, special, tag;
} mapsector_t;
```

Runtime format (`sector_t` in `r_defs.h`):
```c
typedef struct {
    fixed_t floorheight, ceilingheight;
    short floorpic, ceilingpic;           // Flat indices (not names)
    short lightlevel, special, tag;
    int soundtraversed;
    mobj_t* soundtarget;
    int blockbox[4];
    degenmobj_t soundorg;
    int validcount;
    mobj_t* thinglist;
    void* specialdata;
    int linecount;
    struct line_s** lines;
} sector_t;
```

During subsector rendering, the sector's floor/ceiling heights are used to determine visible planes via `R_FindPlane()`.

**Related Files**:  
`doomdata.h`, `r_defs.h`, `p_setup.c` (P_LoadSectors), `r_bsp.c`, `r_plane.c`

**Related Concepts**:  
Subsector, linedef, vertex, flat, plane

---

### Vertex
**In Doom/General**:  
A point in 2D space (x, y coordinate). Vertices are the endpoints of linedefs and segments.

**In This Repo**:  
On-disk format (`mapvertex_t` in `doomdata.h`):
```c
typedef struct {
    short x, y;   // 16-bit fixed-point, 1 unit = 1/16 of a pixel in map coords
} mapvertex_t;
```

Runtime format (`vertex_t` in `r_defs.h`):
```c
typedef struct {
    fixed_t x, y;  // 32-bit fixed-point (upscaled from 16-bit with FRACBITS << shift)
} vertex_t;
```

Segments and linedefs reference vertices. Coordinates are converted from 16-bit WAD format to 32-bit runtime format during loading (P_LoadVertexes).

**Related Files**:  
`doomdata.h`, `r_defs.h`, `p_setup.c` (P_LoadVertexes)

**Related Concepts**:  
Segment, linedef, fixed-point

---

## Spatial Query & Rendering Concepts

### Bounding Box
**In Doom/General**:  
An axis-aligned rectangle (AABB) that encloses a region. Used for visibility testing and culling.

**In This Repo**:  
Represented as an array of 4 fixed_t values:
```c
fixed_t bbox[4];  // [BOXLEFT, BOXRIGHT, BOXTOP, BOXBOTTOM]
```

Bounding boxes in BSP nodes:
```c
node_t->bbox[2][4];  // Two boxes, one for each child
```

Bounding boxes in linedefs (computed during loading):
```c
line_t->bbox[4];
```

Used in visibility culling via `R_CheckBBox()`, which tests if a node's bounding box is visible from the current viewpoint by checking against the current clip list.

**Related Files**:  
`m_bbox.h` (utilities), `r_bsp.c` (R_CheckBBox), `r_defs.h`

**Related Concepts**:  
Frustum clipping, visibility, node

---

### VisPlane (Visibility Plane)
**In Doom/General**:  
A screen-space data structure tracking a unique floor or ceiling plane (identified by height, texture, light level). Used to group pixels for batch rendering of floors/ceilings.

**In This Repo**:  
Defined in `r_defs.h`:
```c
typedef struct {
    fixed_t height;
    int picnum;
    int lightlevel;
    int minx, maxx;
    byte pad1;
    byte top[SCREENWIDTH];
    byte pad2, pad3;
    byte bottom[SCREENWIDTH];
    byte pad4;
} visplane_t;
```

Per subsector, `R_Subsector()` calls `R_FindPlane()` to retrieve or create a visplane for the sector's floor/ceiling. Later, `R_DrawPlanes()` renders all visplanes. Visplanes are derived from BSP subsector sectors.

**Related Files**:  
`r_defs.h`, `r_plane.c` (R_FindPlane), `r_bsp.c` (R_Subsector)

**Related Concepts**:  
Subsector, sector, rendering

---

### DrawSeg (Draw Segment)
**In Doom/General**:  
A screen-space record of a wall segment currently being rendered. Stores 2D projection info, texture scale, and clipping data for sprites.

**In This Repo**:  
Defined in `r_defs.h`:
```c
typedef struct drawseg_s {
    seg_t*     curline;
    int        x1, x2;           // Screen x coordinates
    fixed_t    scale1, scale2, scalestep;
    int        silhouette;       // 0=none, 1=bottom, 2=top, 3=both
    fixed_t    bsilheight, tsilheight;
    short*     sprtopclip, sprbottomclip, maskedtexturecol;
} drawseg_t;
```

Created by `R_StoreWallRange()` (called during segment processing). Array `drawsegs[]` stores all visible segments for the current frame. Used for sprite clipping.

**Related Files**:  
`r_defs.h`, `r_segs.c` (R_StoreWallRange), `r_things.c` (sprite clipping)

**Related Concepts**:  
Segment, rendering, clipping

---

### Clipping / ClipRange
**In Doom/General**:  
Screen-space column tracking. As walls and flats are rendered, a clip list records which screen columns have been filled.

**In This Repo**:  
Defined in `r_bsp.c`:
```c
typedef struct {
    int first, last;
} cliprange_t;

cliprange_t solidsegs[MAXSEGS];
```

Maintained by:
- `R_ClearClipSegs()` — Initialize (screen fully open)
- `R_ClipSolidWallSegment()` — Solid wall blocks screen columns; updates clip list
- `R_ClipPassWallSegment()` — Pass-through wall (two-sided, window) doesn't block

When a segment is fully blocked, rendering skips it.

**Related Files**:  
`r_bsp.c`, `r_segs.c`

**Related Concepts**:  
Segment, rendering, visibility

---

### Blockmap
**In Doom/General**:  
A 2D spatial hash grid overlaid on the map. Used to accelerate collision detection by partitioning moving objects and walls into fixed-size blocks.

**In This Repo**:  
Loaded from BLOCKMAP lump in `p_setup.c` (P_LoadBlockMap). Described in `p_local.h`:
```c
#define MAPBLOCKUNITS   128
#define MAPBLOCKSIZE    (MAPBLOCKUNITS * FRACUNIT)
#define MAPBLOCKSHIFT   (FRACBITS + 7)
```

Blockmap is independent of BSP but complementary. Used in collision queries (p_map.c, p_maputl.c) to iterate only potentially colliding objects/walls, rather than testing all.

**Related Files**:  
`p_setup.c` (P_LoadBlockMap), `p_map.c`, `p_local.h`

**Related Concepts**:  
Spatial subdivision, collision

---

### REJECT Matrix
**In Doom/General**:  
A lookup table (bit array) recording which sector pairs can potentially see each other. Used to quickly reject line-of-sight checks between distant sectors.

**In This Repo**:  
Loaded from REJECT lump in `p_setup.c`. Consulted by sight-checking code (p_sight.c) to skip expensive ray-casting for known-invisible sector pairs. Separate from BSP but supports gameplay visibility logic.

**Related Files**:  
`p_setup.c` (P_LoadReject), `p_sight.c`

**Related Concepts**:  
Sector, visibility

---

## Notes

- **Fixed-Point Representation**: Map coordinates use 16-bit fixed-point on disk and 32-bit fixed-point at runtime. `FRACBITS=16`, so `value << FRACBITS` converts an integer to fixed-point.
- **Angle Representation**: Angles use `angle_t` (typically `unsigned int`), where 2^32 represents a full 360°.
- **Texture Names to Indices**: Texture names (like "STONE1") are converted to indices during loading (R_FlatNumForName, R_TextureNumForName).
- **Structure Alignment**: Runtime structures are carefully laid out to minimize padding; some structures pre-allocate pointer arrays for efficiency.
