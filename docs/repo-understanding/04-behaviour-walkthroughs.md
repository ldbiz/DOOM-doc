# 04 - BSP Behaviour Walkthroughs

## Behaviour 1: Loading BSP/map lumps from WAD data

**Trigger**: Map change — `P_SetupLevel()` is called with a map lump name (e.g., `"E1M1"`).

**Files involved**: `p_setup.c` (all loaders), `w_wad.c` (lump caching), `doomdata.h` (on-disk types), `m_fixed.h` (coordinate scaling), `m_swap.h` (endian handling), `r_defs.h` (runtime types).

**Bird's-eye flow**:

1. `W_GetNumForName(lumpname)` returns the starting lump index for the map.
2. In order: `P_LoadBlockMap()` → `P_LoadVertexes()` → `P_LoadSectors()` → `P_LoadSideDefs()` → `P_LoadLineDefs()` → `P_LoadSubsectors()` → `P_LoadNodes()` → `P_LoadSegs()`.
3. Each loader calls `W_CacheLumpNum(lumpnum + ML_*, PU_STATIC)`, casts the raw bytes to the corresponding `map*_t*` type, iterates entries, converts 16-bit values to 32-bit fixed-point via `SHORT(value) << FRACBITS`, and stores in the runtime array.
4. `P_LoadNodes()` specifically: partition line `(x,y,dx,dy)` and both `bbox[2][4]` are scaled `<<FRACBITS`; `children[2]` are copied as-is (preserving `NF_SUBSECTOR` high bits).
5. `P_LoadSubsectors()` stores `numlines`/`firstline` as direct indices into the `segs[]` array.
6. `P_LoadSegs()` links each seg to its `vertexes[]`, `lines[]`, `sides[]`, and sets `frontsector`/`backsector` pointers.
7. `P_GroupLines()` post-processes: assigns `subsector->sector`, builds per-sector `lines[]` arrays.
8. REJECT lump is cached directly into `rejectmatrix` (no loader function).

**Inputs**: WAD file handle + map lump name string.

**Outputs**: Global arrays `nodes[]`, `subsectors[]`, `segs[]`, `vertexes[]`, `lines[]`, `sides[]`, `sectors[]` populated; globals `numnodes`, `numsubsectors`, `numsegs`, etc. set.

**Side effects**: Zone memory allocated (`PU_LEVEL` tag). Temporary lump cache buffers freed via `Z_Free()`.

---

## Behaviour 2: Rendering traversal — locating viewpoint in BSP and producing draw order

**Trigger**: Every rendered frame — `R_RenderPlayerView()` (`r_main.c`).

**Files involved**: `r_main.c` (setup), `r_bsp.c` (traversal), `r_segs.c` (seg clipping), `r_plane.c` (visplanes), `r_things.c` (sprites), `r_defs.h` (types), `r_state.h` (shared state).

**Bird's-eye flow**:

1. **Frame setup**: `R_SetupFrame()` sets `viewx`, `viewy`, `viewz`, `viewangle` from the player mobj. Computes `viewsin`, `viewcos`, and derived projection tables.
2. **Clear transient state**: `R_ClearClipSegs()` (solidsegs), `R_ClearDrawSegs()` (ds_p = drawsegs), `R_ClearPlanes()` (floorclip/ceilingclip), `R_ClearSprites()` (vissprite_p).
3. **Traversal**: `R_RenderBSPNode(numnodes - 1)` recursively walks from root:
   - If `bspnum & NF_SUBSECTOR` → `R_Subsector(bspnum & ~NF_SUBSECTOR)` — process leaf.
   - Otherwise: get `node = &nodes[bspnum]`, determine `side = R_PointOnSide(viewx, viewy, node)`.
   - Recurse into `children[side]` (front child — the side containing the viewpoint).
   - If `R_CheckBBox(node->bbox[side^1])` is visible → recurse into `children[side^1]` (back child).
   - This yields **front-to-back traversal**: nearer subsectors processed first.
4. **Leaf processing** (`R_Subsector`):
   - Sets `frontsector = sub->sector`.
   - If floor below view: `floorplane = R_FindPlane(floorheight, floorpic, lightlevel)`.
   - If ceiling above view or sky: `ceilingplane = R_FindPlane(...)`.
   - `R_AddSprites(frontsector)` — projects all things in sector into `vissprites[]`.
   - For each seg in subsector: `R_AddLine(seg)` — computes screen range, clips, creates drawseg.
5. **Seg processing** (`R_AddLine` → `r_segs.c`):
   - Compute angles from viewpoint, map to screen X coordinates, backface-cull.
   - For solid (one-sided) segs: `R_ClipSolidWallSegment()` → merges into `solidsegs[]`, calls `R_StoreWallRange()` for visible fragments.
   - For passable (two-sided window) segs: `R_ClipPassWallSegment()` → finds visible gaps, calls `R_StoreWallRange()`.
   - `R_StoreWallRange()`: creates `drawseg_t` entry at `ds_p++`, computes scale values, determines textures (top/mid/bottom/masked), calls `R_CheckPlane()` to register visplanes, calls `R_RenderSegLoop()` to draw wall columns and update `floorclip[]`/`ceilingclip[]`.
6. **Post-traversal**:
   - `R_DrawPlanes()` — batches all visplanes into horizontal spans and draws them.
   - `R_DrawMasked()` — sorts vissprites by distance, draws back-to-front with per-column clipping against drawsegs.

**Inputs**: Player position/orientation (`viewx`, `viewy`, `viewz`, `viewangle`), BSP tree (`nodes[]`, `subsectors[]`, `segs[]`).

**Outputs**: Framebuffer pixels. Transient arrays: `drawsegs[]`, `visplanes[]`, `vissprites[]`, `solidsegs[]`, `floorclip[]`/`ceilingclip[]`.

**Side effects**: `validcount` incremented for sector/line deduplication.

---

## Behaviour 3: Line-of-sight check (monster AI visibility)

**Trigger**: Monster AI decides whether it can see the player — `P_CheckSight(t1, t2)`.

**Files involved**: `p_sight.c` (sight logic), `p_setup.c` (REJECT loading), `r_main.c` (side test), `r_defs.h` (BSP types).

**Bird's-eye flow**:

1. **Coarse REJECT check**: Compute sector indices `s1`, `s2` for the two mobjs. Calculate `pnum = s1 * numsectors + s2`. If `rejectmatrix[pnum >> 3] & (1 << (pnum & 7))` — immediately return `false` (no LOS possible).
2. **BSP traversal**: If not rejected, set up a trace from `t1` to `t2`. Call `P_CrossBSPNode(numnodes - 1)`.
3. **Node checking**: `P_CrossBSPNode()` determines which side of each partition line the trace endpoints lie on, recurses into intersected subtrees.
4. **Subsector checking** (`P_CrossSubsector()`): For each subsector the trace passes through, iterates its segs:
   - Computes whether the trace crosses each seg.
   - For two-sided lines: adjusts `open` range based on sector heights.
   - If `open` range closes to zero → sight is blocked → return `false`.
5. **Result**: If traversal completes without the open range closing, return `true` (visible).

**Inputs**: Two `mobj_t` pointers.

**Outputs**: Boolean (visible/occluded).

**Side effects**: None on game state. Uses `validcount` for deduplication.

---

## Behaviour 4: Point-in-subsector lookup (spatial query)

**Trigger**: `P_SetThingPosition()` when a mobj is spawned or moves — needs to know which subsector contains the mobj.

**Files involved**: `r_main.c` (`R_PointInSubsector`), `p_maputl.c` (`P_SetThingPosition`), `r_bsp.c` (side test).

**Bird's-eye flow**:

1. `R_PointInSubsector(x, y)` starts at root: `nodenum = numnodes - 1`.
2. While `!(nodenum & NF_SUBSECTOR)`:
   - `node = &nodes[nodenum]`
   - `side = R_PointOnSide(x, y, node)` — determines which side of the partition line (x,y) falls on.
   - `nodenum = node->children[side]`
3. When a subsector is reached: return `&subsectors[nodenum & ~NF_SUBSECTOR]`.
4. Special case: if `numnodes == 0` (single-subsector map), returns `subsectors` directly.

**Inputs**: `fixed_t x, y` (world coordinates).

**Outputs**: `subsector_t*` pointer.

**Side effects**: None.

---

## Behaviour 5: Wall occlusion clipping during BSP traversal

**Trigger**: Each seg processed during `R_Subsector()` in rendering traversal.

**Files involved**: `r_bsp.c` (`R_AddLine`, `R_ClipSolidWallSegment`, `R_ClipPassWallSegment`), `r_segs.c` (`R_StoreWallRange`, `R_RenderSegLoop`).

**Bird's-eye flow**:

1. `R_AddLine(seg)` computes the seg's screen X range `[x1, x2]` and classifies it as solid (one-sided/closed door) or passable (window/two-sided with opening).
2. **Solid segs**: `R_ClipSolidWallSegment(first, last)` walks the `solidsegs[]` clip range list:
   - Finds fragments of `[first, last]` that are not already covered by earlier (closer) solid walls.
   - For each visible fragment: calls `R_StoreWallRange()` to create a drawseg and draw the wall.
   - Inserts the new range into `solidsegs[]` (merging with adjacent ranges), so later (farther) segs are clipped by it.
3. **Passable segs**: `R_ClipPassWallSegment(first, last)` walks `solidsegs[]` to find visible gaps, calls `R_StoreWallRange()` for each, but does NOT modify `solidsegs[]` (windows don't occlude).
4. **Effect**: Front-to-back BSP traversal + solidsegs insertion = automatic painter's-algorithm-like occlusion. Walls closer to the camera block walls behind them.

**Inputs**: `seg_t*` line segment from BSP subsector.

**Outputs**: `drawseg_t` entries in `drawsegs[]`, updated `solidsegs[]`, updated `floorclip[]`/`ceilingclip[]`, visplane top/bottom per-column data.

**Side effects**: Framebuffer modified (wall columns drawn). `ds_p` incremented.
