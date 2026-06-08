# Change Map: Guide for Making BSP-Related Changes

This section provides a roadmap for modifying BSP-related behavior. For each category of change, the relevant files and caveats are listed.

---

## If I need to change map/WAD lump loading

**Likely files to modify**:
- `p_setup.c` — P_LoadVertexes, P_LoadLineDefs, P_LoadSideDefs, P_LoadSectors, P_LoadSegs, P_LoadSubsectors, P_LoadNodes, P_LoadBlockMap, P_LoadReject
- `doomdata.h` — On-disk structure definitions (mapvertex_t, mapnode_t, etc.)
- `w_wad.c` — Low-level WAD I/O (only if changing how lumps are fetched)

**Why these files matter**:
- Each `P_Load*` function deserializes one lump type and converts to runtime format.
- If you change the on-disk format, you must update the corresponding structure in `doomdata.h` and the load function.
- WAD I/O is abstracted in `w_wad.c`; changes there affect all lump loading.

**Caveats**:
- Lump order is fixed (defined in `doomdata.h` as ML_* enum). Dependencies between lumps must be respected (e.g., segs must load after vertices).
- Changes to on-disk format break compatibility with existing WAD files. Consider versioning or detection.
- Cross-references between structures (e.g., seg→linedef pointers) must be re-established during load if format changes.
- All coordinates are upscaled from 16-bit WAD to 32-bit runtime; if you change coordinate precision, both files need updates.

---

## If I need to change BSP node/subsector representation

**Likely files to modify**:
- `r_defs.h` — node_t, subsector_t structure definitions
- `p_setup.c` — P_LoadNodes, P_LoadSubsectors (conversion from mapnode_t, mapsubsector_t)
- `r_bsp.c` — BSP traversal logic using node_t and subsector_t fields

**Why these files matter**:
- `r_defs.h` defines the runtime representation; all code that accesses nodes/subsectors depends on these definitions.
- `p_setup.c` converts WAD format to runtime format; if you change runtime format, the conversion must also change.
- `r_bsp.c` directly accesses node fields (x, y, dx, dy, children[], bbox[]); any change must be reflected here.

**Caveats**:
- node_t and subsector_t are tightly coupled to the partition line testing and traversal logic. Changes to fields (e.g., adding/removing bbox) require updates throughout `r_bsp.c` and `r_main.c`.
- The NF_SUBSECTOR flag is baked into the children[] index encoding. If you change this, all node references must change.
- subsector_t references segs[] by index (firstline, numlines). If you change this, segment registration in `R_Subsector` must change.
- Pointer arithmetic and alignment assumptions are implicit. Changing struct layout may affect performance or require padding adjustments.

---

## If I need to change traversal order

**Likely files to modify**:
- `r_bsp.c` — R_RenderBSPNode() function
- `r_main.c` — R_PointOnSide() (if you change which side is "front")
- `r_segs.c` — Clipping logic (if traversal order affects visibility)

**Why these files matter**:
- `R_RenderBSPNode()` is the core recursion; front-to-back order is implemented here (front child visited first, back child visited conditionally).
- `R_PointOnSide()` determines front vs. back; if you change the test, traversal order changes.
- Clipping assumes front-first traversal; changing order may require clipping logic overhaul.

**Caveats**:
- Traversal order is *fundamental* to rendering correctness. Changing it requires careful consideration of visibility and clipping.
- The current design (front-first, back conditional) minimizes overdraw. Alternative orders may be slower.
- Bounding box culling assumes front-first order. If you change order, visibility checks may need revision.
- Screen space clipping (solidsegs[]) is built incrementally during traversal. If traversal order changes, clipping must be updated to match.

---

## If I need to change wall rendering from BSP traversal

**Likely files to modify**:
- `r_segs.c` — R_AddLine, R_StoreWallRange, segment classification logic
- `r_bsp.c` — R_Subsector, which calls R_AddLine
- `r_defs.h` — drawseg_t structure (if you change what data is recorded per segment)

**Why these files matter**:
- `r_segs.c` contains all wall rendering logic: segment clipping, texture determination, wall type classification.
- `R_AddLine()` bridges BSP traversal and rendering by processing each segment.
- `R_StoreWallRange()` records wall rendering orders; changes here affect all wall output.

**Caveats**:
- Wall rendering depends on correct clipping (solidsegs[]). Changes to rendering may break clipping assumptions.
- Wall visibility is determined by sector geometry (front/back heights, textures). Changing classification logic may cause visual errors (incorrect windows, missing walls).
- Texture coordinates and lighting are baked into wall processing. Changes require careful math to preserve visual correctness.
- Multiple walls may map to the same screen region. Rendering order must be correct to avoid visible flicker or clipping errors.

---

## If I need to change visibility/clipping behaviour

**Likely files to modify**:
- `r_bsp.c` — R_CheckBBox (bounding box frustum culling)
- `r_segs.c` — R_ClipSolidWallSegment, R_ClipPassWallSegment (screen space clipping)
- `r_main.c` — Angle-to-screen coordinate conversion (viewangletox[], xtoviewangle[])

**Why these files matter**:
- `R_CheckBBox()` performs early-exit frustum culling; if a node is off-screen, its entire subtree is skipped.
- `R_ClipSolidWallSegment()` and `R_ClipPassWallSegment()` manage screen-space coverage; they determine what actually gets rendered.
- Angle-to-screen mapping is critical for clipping decisions; errors here cause incorrect visibility.

**Caveats**:
- Clipping is an optimization; broken clipping results in either excessive rendering (slowness) or missing geometry (visual errors).
- Bounding box tests use fixed-point arithmetic; precision is important. Rounding errors can cause false rejections or false inclusions.
- Screen-space clipping uses a sorted list (solidsegs[]) for efficiency. Changing the algorithm may degrade performance if the new approach is O(n²) instead of O(n).
- Angle wrapping (2^32 wraparound) is implicit in angle arithmetic; changes must account for this.

---

## If I need to change collision or spatial queries

**Likely files to modify**:
- `p_map.c` — P_CheckPosition, P_PathTraverse, collision detection logic
- `p_maputl.c` — Blockmap iteration utilities
- `p_local.h` — MAPBLOCK* constants and defines

**Why these files matter**:
- Collision detection does *not* directly use the BSP tree; it uses the blockmap (spatial hash).
- `p_map.c` implements high-level collision tests; `p_maputl.c` provides blockmap utilities.
- MAPBLOCK* constants define the grid size; changes here affect spatial partitioning precision.

**Caveats**:
- Collision is independent of rendering, but uses related structures (sectors, lines, sidedefs).
- Blockmap is orthogonal to BSP; changing collision doesn't require BSP changes (and vice versa).
- Blockmap size (128 units per block) is a performance tuning parameter. Smaller blocks = more precision but slower iteration; larger blocks = fewer iterations but coarser spatial partitioning.
- Thing collision radius (PLAYERRADIUS) affects which blocks are queried; changing this affects collision behavior.

---

## If I need to add BSP debug visualisation

**Likely files to modify**:
- `r_bsp.c` — Add debug output or visualization logic
- `r_draw.c` — Add drawing functions for debug output
- New file: `r_bspvis.c` (if creating substantial debug module)

**Why these files matter**:
- `r_bsp.c` is where traversal happens; adding debug output here shows tree traversal order.
- `r_draw.c` provides low-level drawing primitives; use these for visualizing geometry.
- A separate debug module keeps production code clean.

**Caveats**:
- Debug visualization may have significant performance overhead; disable in release builds.
- Visualizing the BSP tree (as lines) requires converting world coordinates to screen space; use existing angle-to-screen functions.
- Visualizing traversal order (coloring subsectors by depth) is easier than visualizing the tree structure itself.
- Be careful not to break rendering while adding debug code; test thoroughly.

---

## If I need to add or update BSP tests/fixtures

**Likely files to create**:
- New test map WAD files (using external editors)
- Optional: Test harness in `test_bsp.c` (if adding unit tests)

**Why tests matter**:
- The codebase has no formal tests. Adding tests improves regression detection.
- Manual test maps (WAD files) can be created and checked in to document BSP behavior.

**Caveats**:
- This codebase was not designed with testing in mind. Adding unit tests requires refactoring (decoupling code from globals).
- Test maps require external tools (map editors) to create; they are not part of the source tree.
- If you add C unit tests, you'll need to link against or refactor significant portions of the rendering and collision code.
- Minimal viable approach: create test maps, document them, and manually verify correctness.

---

## Quick Reference: File Responsibilities

| Task | Primary File | Secondary Files |
|------|--------------|-----------------|
| Load BSP data | p_setup.c | w_wad.c, doomdata.h, r_defs.h |
| Traverse BSP tree | r_bsp.c | r_main.c (R_PointOnSide) |
| Clip walls to screen | r_segs.c | r_bsp.c (R_Subsector) |
| Frustum cull nodes | r_bsp.c (R_CheckBBox) | — |
| Spatial queries | p_map.c | p_maputl.c, p_local.h |
| Point-on-side test | r_main.c (R_PointOnSide) | — |
| Draw planes | r_plane.c | r_bsp.c (R_Subsector) |
| Draw sprites | r_things.c | r_bsp.c (R_Subsector, R_AddSprites) |

---

## Common Pitfalls

1. **Changing node_t without updating traversal**: If you add/remove fields, `r_bsp.c` and `p_setup.c` must be updated.
2. **Breaking lump order dependencies**: If you reorder lumps, cross-references fail and maps don't load.
3. **Assuming front/back traversal order**: Some optimizations depend on front-first order; changing it breaks them.
4. **Off-by-one errors in clipping**: Clipping logic is subtle; changes often introduce rendering artifacts.
5. **Ignoring fixed-point precision**: Coordinates are fixed-point; mixing with integers causes bugs.
6. **Not testing with complex maps**: Simple maps may work, but complex ones expose edge cases.

---

## Recommended Approach for Changes

1. **Understand the current code**: Read `r_bsp.c` and `r_main.c` thoroughly.
2. **Plan changes carefully**: Document which functions/structures will be affected.
3. **Test extensively**: Use a variety of maps (E1M1–E1M9) to verify correctness.
4. **Check for performance regression**: Measure frame rate before and after; some changes may be slower than alternatives.
5. **Review cross-file impacts**: Changes in one file often require changes in others; trace all dependencies.
