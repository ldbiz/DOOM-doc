# Configuration & Environment

## Hardcoded BSP-Related Configuration

This codebase contains minimal explicit configuration for BSP. Most settings are hardcoded constants or derived from map lumps. The following are the relevant hardcoded values and assumptions:

### Coordinate System

**Fixed-Point Precision**:
- **On-disk (WAD)**: 16-bit signed integer representing map units. Resolution: 1 pixel / 16 = 1/16 pixel precision.
- **Runtime**: 32-bit fixed-point (`fixed_t`), upscaled by `<< FRACBITS` (where `FRACBITS = 16`).
  - Example: WAD vertex at x=512 becomes runtime x = 512 << 16 = 33554432 (fixed_t).
- **Geometry precision**: 1 unit in fixed-point ≈ 1 map unit ≈ 1/16 pixel visual scale (depends on aspect ratio).

**Files**: `m_fixed.h` (FRACBITS definition), `p_setup.c` (load-time conversion)

### Angle Representation

**Format**: `angle_t` (typically `unsigned int` / `uint32_t`).
- Full 360° = 2^32 = 0x100000000 (wraps around).
- 45° = 0x20000000 (2^32 / 8).
- Used throughout: `viewangle`, partition line angles, segment angles.

**Fine Angles**: `FINEANGLES` = 8192 (2^13 subdivisions of a full rotation for lookup table efficiency).
- `finesine[]`, `finecosine[]` precomputed tables.

**Files**: `tables.c`, `doomdef.h`

### Screen and Rendering

**Default Screen Dimensions**:
```c
#define SCREENWIDTH  320
#define SCREENHEIGHT 200
```

**View Field of View**:
```c
#define FIELDOFVIEW 2048
```
(In fineangles; ~2048/8192 ≈ 90° horizontal FOV)

**Maximum Draw Segments**:
```c
#define MAXDRAWSEGS 256
```
(Limit on simultaneous visible wall segments)

**Clip Segments Limit**:
```c
#define MAXSEGS 32
```
(Maximum number of clip ranges during screen clipping)

**Files**: `doomdef.h`, `r_bsp.c`

### Movement & Collision

**Map Block Size** (for spatial hashing):
```c
#define MAPBLOCKUNITS    128     // Map units per block
#define MAPBLOCKSIZE     (MAPBLOCKUNITS * FRACUNIT)
#define MAPBLOCKSHIFT    (FRACBITS + 7)
```
(Blocks are 128 × 128 map units)

**Player Radius**:
```c
#define PLAYERRADIUS (16 * FRACUNIT)
```
(Player's collision radius; affects blockmap cell queries)

**Files**: `p_local.h`, `p_setup.c`

### Lump Ordering

**Lump Enumeration** (from `doomdata.h`):
```c
enum {
  ML_LABEL,      // 0: Map name
  ML_THINGS,     // 1: Entities
  ML_LINEDEFS,   // 2: Walls/triggers
  ML_SIDEDEFS,   // 3: Wall textures/sectors
  ML_VERTEXES,   // 4: Vertex coordinates
  ML_SEGS,       // 5: Wall segments (BSP split)
  ML_SSECTORS,   // 6: Subsectors (BSP leaves)
  ML_NODES,      // 7: BSP nodes
  ML_SECTORS,    // 8: Floor/ceiling/light
  ML_REJECT,     // 9: Sight rejection matrix
  ML_BLOCKMAP    // 10: Spatial grid
};
```

**Load Order Constraints**: Lumps must be loaded in the order defined above. Cross-references between structures depend on earlier lumps being loaded first.

**Files**: `doomdata.h`, `p_setup.c` (P_SetupLevel)

### Limits

| Constant | Value | Purpose |
|----------|-------|---------|
| MAXDRAWSEGS | 256 | Max simultaneous visible segments |
| MAXSEGS | 32 | Max clip ranges |
| MAX_DEATHMATCH_STARTS | 10 | Deathmatch spawn points |
| MAXPLAYERS | 4 | Multiplayer limit |
| MAXLINODES | 65536 | Max linedef/node count (approximately) |

**Files**: `doomdef.h`, `r_bsp.h`, `p_local.h`

### BSP Tree Properties

**Root Node Index**:
```c
R_RenderBSPNode(numnodes - 1);  // Root is always the last node
```
(BSP trees are built so the root is at index `numnodes - 1`)

**Subsector Marker**:
```c
#define NF_SUBSECTOR 0x8000  // High bit marks subsector leaf
```

**Files**: `doomdata.h`, `r_bsp.c`

### Plane Caching

**Max Simultaneous Planes**:
```c
#define MAXVISPLANES 128  // (from r_plane.c)
```
(Cached floor/ceiling planes per frame)

**Files**: `r_plane.c`

---

## Environment Variables & Command-Line Flags

The codebase accepts command-line arguments for debugging and tuning, but none specifically affect BSP-related behavior beyond general rendering.

**Potentially Relevant**:
- `-width` / `-height` — Screen resolution (affects FOV, clipping)
- `-devparm` — Developer mode; enables additional logging
- `-skill` — Difficulty level (affects monster spawning, which uses the BSP to determine location)

**Files**: `d_main.c`, `m_argv.c`

---

## Map/WAD Selection

Maps are selected by episode and map number in the registered DOOM game mode:
```c
// In g_game.c
G_InitNew(skill, episode, map);  // E.g., skill=3, episode=1, map=1 → E1M1
```

The map name (e.g., "E1M1" or "MAP01" in DOOM II) is used to find the map lump in the WAD.

**Files**: `g_game.c`, `w_wad.c`

---

## Assumptions About Doom Engine Behavior

The following assumptions are baked into the code:

1. **BSP trees are pre-built**: The repository does not include a BSP builder. Trees are created offline by tools (WadAuthor, ZenNode, etc.) and embedded in WADs. The codebase only *loads* and *traverses* them.

2. **Root node is last**: BSP trees are constructed so the root is at index `numnodes - 1`.

3. **Partition lines are from linedef segments**: BSP builder uses actual map linedefs (or their extensions) as partition lines.

4. **Subsectors form convex regions**: All geometry within a subsector belongs to one sector and is convex (no holes).

5. **Segments are continuous within subsectors**: Segments in a subsector form a contiguous list in the segs[] array; all reference the same sector.

6. **Blockmap is mandatory**: Collision and spatial queries assume blockmap exists. (Some ports add blockmap generation if missing.)

7. **Fixed-point overflow handling**: Code uses `FixedMul()` and `FixedDiv()` for arithmetic to avoid overflow, but does not check for it. Assumes well-formed maps.

8. **Angles wrap at 2^32**: Angle arithmetic does not explicitly check for wraparound; it relies on unsigned overflow.

---

## Debugging & Visualization

The codebase includes no built-in BSP visualization or debugging output, but the following flags/modes may be useful:

**Visible in code**:
- `validcount` (in `r_main.c`) — Used to avoid redundant processing; incremented each frame
- `sscount`, `linecount`, `loopcount` — Statistics for profiling (initialized but not always used)
- `framecount` — Frame counter for debugging

**Related files**: `r_main.c`, `doomstat.c`

---

## No Data-Driven Configuration

Unlike modern engines, this codebase has **no configuration files** for BSP or rendering. All behavior is either:
- **Hardcoded constants** (compiled in)
- **WAD-embedded data** (map lumps)
- **Command-line arguments** (very limited)

This is by design—the original DOOM was built for limited 1990s systems with minimal overhead.

---

## Notes on Map Format Assumptions

- **Coordinate Scale**: All map coordinates are in 16-bit integers on disk. No support for "extended" formats or 32-bit coordinates.
- **Texture Naming**: Texture names are 8-character null-padded strings. No support for long names.
- **Lump Ordering**: The lump enumeration in `doomdata.h` is strict. Maps must follow this order.
- **Linedef/Sidedef Linking**: Linedefs reference sidedefs by index; if indices are out of bounds, undefined behavior.

These are fundamental assumptions in the WAD format and cannot be changed without breaking compatibility.
