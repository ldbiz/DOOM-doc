# Caveats and Unknowns

This document lists BSP-related topics that could not be fully confirmed from the repository or that have genuine ambiguity or missing context.

---

## Data Format Ambiguities

### BSP Builder Algorithm Not Included
**Status**: Unknown by inspection  
**Issue**: The repository contains only the BSP *traversal* and *rendering* code, not the BSP *builder* (the algorithm that creates the tree from raw linedef data). The comments suggest BSP trees are pre-built offline.

**Questions**:
- What algorithm is used? (Classic BSP builder, BRute-force, Doom builder algorithm?)
- How are partition lines selected? (Heuristic based, balanced, or some other criteria?)
- Are there any known degenerate cases (e.g., pathological tree depths)?

**Impact**: Cannot verify whether BSP creation is optimal or if the tree structure could be improved.

---

### Root Node Index Assumption
**Status**: Assumed, not explicitly validated  
**Issue**: Code assumes the root node is always at index `numnodes - 1`. This is not verified at load time.

**Questions**:
- Is this guaranteed by the BSP builder or WAD format?
- What happens if a WAD file has a different root index?
- Should there be a safety check?

**Impact**: Malformed WAD files with non-standard root indices would cause incorrect rendering or crashes.

---

### Subsector Ordering
**Status**: Assumed but not documented  
**Issue**: The code assumes segs within a subsector are contiguous in the segs[] array. This is stored as `firstline` and `numlines` indices.

**Questions**:
- Is contiguity guaranteed by the BSP builder?
- What if a BSP builder outputs non-contiguous segs for a subsector?

**Impact**: Non-contiguous segs would require changes to subsector processing (currently a simple linear loop).

---

## Rendering Logic Ambiguities

### "Empty Line" Detection
**Status**: Partially understood  
**Issue**: Lines with identical floor/ceiling heights, light levels, and no middle texture are skipped entirely. The code treats them as "empty" (triggers or special lines with no visual geometry).

**Questions**:
- Are these *guaranteed* to have no visual geometry?
- Could a map legitimately need to render such a line (e.g., for transparency tricks)?
- How do these lines interact with sector special effects?

**Impact**: Mods or edge cases might require rendering these lines; the current skip logic might cause visual glitches.

---

### Backface Culling Precision
**Status**: Implicit in angle arithmetic  
**Issue**: Backface culling checks if `span >= ANG180` (180°). Angles use modular arithmetic (2^32 = 360°).

**Questions**:
- What about lines exactly perpendicular to the view? (span == ANG90)
- Does angle wrapping ever cause precision issues near 0° or 360°?
- Has this been tested with pathological camera angles?

**Impact**: Certain edge cases (exact perpendicularity, extreme angles) might produce artifacts.

---

### Visplane Caching Strategy
**Status**: Visible but not fully documented  
**Issue**: `R_FindPlane()` caches floor/ceiling planes and re-uses them if height/texture/light match. The cache has a max size (MAXVISPLANES, typically 128).

**Questions**:
- What happens when the cache is full?
- Is there an eviction strategy, or does rendering fail?
- Can a single frame exceed 128 unique planes legitimately?

**Impact**: Complex maps or unusual lighting scenarios might cause plane cache overflow; unclear if this is handled gracefully.

---

## Collision & Spatial Query Unknowns

### Blockmap Generation Not Included
**Status**: WAD format documented, but generation not included  
**Issue**: Like BSP trees, blockmaps are pre-built and embedded in WAD files. The repository does not contain blockmap generation code.

**Questions**:
- If a WAD file lacks a blockmap, does the engine handle it gracefully?
- Can a blockmap be corrupted or malformed?

**Impact**: Certain mods or tools might create WAD files without proper blockmaps; unclear if the engine detects and recovers.

---

### REJECT Matrix Handling
**Status**: Loaded but usage not fully visible  
**Issue**: The reject matrix is loaded but only referenced in `p_sight.c` (line-of-sight checking). The actual LOS logic is not BSP-specific.

**Questions**:
- How accurate is the reject matrix? (Some BSP builders create conservative matrices.)
- What if the matrix is corrupt or missing?
- Can LOS be correct if the reject matrix is wrong?

**Impact**: Sight checks might be inefficient or incorrect if the reject matrix is malformed.

---

### Collision with Multiple Objects
**Status**: Blockmap-based, BSP role unclear  
**Issue**: Moving objects test collision against lines and other objects using the blockmap. The BSP tree is not directly involved, but bounding boxes (from BSP nodes) may be used.

**Questions**:
- Are BSP bounding boxes consulted during collision?
- Could the collision system be accelerated further using BSP?

**Impact**: Collision performance might be suboptimal compared to a full BSP-based approach.

---

## Fixed-Point & Coordinate Precision

### 16-bit WAD Precision
**Status**: Documented in code, but limits unclear  
**Issue**: WAD coordinates use 16-bit signed integers (-32768 to 32767 map units). This is upscaled to 32-bit runtime format.

**Questions**:
- What is the intended scale? (1 unit = 1 pixel? 1/8 pixel? Doom-specific?)
- Can maps exceed 32-bit signed range in runtime?
- Has any map come close to the 16-bit limits?

**Impact**: Very large maps might hit coordinate limits; unclear how they're handled.

---

### Fixed-Point Overflow in Calculations
**Status**: FixedMul and FixedDiv used, but overflow not handled  
**Issue**: `FixedMul()` and `FixedDiv()` are used throughout, but overflow is not explicitly checked.

**Questions**:
- Can extreme partition lines or viewpoints cause overflow?
- What happens if FixedMul overflows (wraps around)?
- Have degenerate maps been tested for this?

**Impact**: Pathological maps might trigger numeric overflow, causing incorrect rendering or crashes.

---

## Performance & Optimization Unknowns

### BSP Tree Depth Not Analyzed
**Status**: Not documented or validated  
**Issue**: Tree depth affects recursion depth in `R_RenderBSPNode()`. Deep trees could cause stack overflow.

**Questions**:
- What is the typical tree depth for DOOM maps? (10? 20? 40?)
- What is the maximum observed depth?
- Is there a hard stack limit or guard?

**Impact**: Deeply balanced trees could exceed stack limits on low-memory systems.

---

### Clipping Algorithm Efficiency
**Status**: Implemented but worst-case complexity unclear  
**Issue**: The `solidsegs[]` clipping list uses insertion and merging. Complexity depends on fragment count.

**Questions**:
- What is the worst-case number of fragments?
- How does performance scale with complex geometry?
- Could the clipping be optimized (e.g., using binary trees)?

**Impact**: Complex scenes might have clipping performance issues; alternatives not explored.

---

### Sprite Sorting
**Status**: Sprites are sorted per subsector but global order unclear  
**Issue**: Sprites are collected during BSP traversal and sorted by distance. The sorting algorithm and stability are not clear.

**Questions**:
- How are sprites at the same distance ordered?
- Could sprites sorting produce visual anomalies with multiple sprites at the same position?
- Is the sorting algorithm stable?

**Impact**: Rare cases (multiple sprites overlapping) might have undefined visual order.

---

## Missing Context & Documentation

### Angle-to-Screen Mapping
**Status**: Tables precomputed, but details sparse  
**Issue**: `viewangletox[]` and `xtoviewangle[]` are precomputed lookup tables, but the exact mapping logic is not obvious.

**Questions**:
- How is the mapping computed? (Trigonometric? Lookup-based? Approximated?)
- What is the precision? (Rounding errors?)
- Is there any distortion? (Rectilinear projection vs. other?)

**Impact**: Port to different display resolutions might have visual artifacts if mapping assumptions are violated.

---

### Sound Source Localization
**Status**: Referenced but not analyzed  
**Issue**: Sectors have a `degenmobj_t soundorg` and sound target tracking (`soundtraversed`, `soundtarget`). Sound logic is in `s_sound.c`, not examined.

**Questions**:
- How does the sound system interact with BSP?
- Could incorrect BSP traversal affect sound? (e.g., sounds appearing from wrong direction)

**Impact**: Sound bugs might be BSP-related but are hard to diagnose without sound code analysis.

---

### Automap (`am_map.c`)
**Status**: Present but not analyzed  
**Issue**: The automap renders the map in 2D. It likely uses BSP/map data to draw lines.

**Questions**:
- Does the automap duplicate rendering logic or reuse BSP data?
- Are there automap-specific bugs related to BSP?
- Can automap reveal BSP tree structure visually?

**Impact**: Automap bugs might reveal BSP issues; conversely, BSP changes might break automap.

---

### Network/Multiplayer Interactions
**Status**: Multiplayer code exists (`d_net.c`) but not analyzed  
**Issue**: Multiplayer affects how map data is shared and how BSP is traversed (per-player viewpoints).

**Questions**:
- How is BSP traversal synchronized in multiplayer?
- Could network lag cause BSP-related desyncronization?
- Are there multiplayer-specific BSP edge cases?

**Impact**: Multiplayer bugs might be BSP-related but hard to reproduce.

---

## Potential Regression Risks

### High-Risk Changes
These areas have the most potential for subtle bugs if modified:

1. **R_PointOnSide()** — Core to BSP traversal; errors cause incorrect rendering order
2. **R_ClipSolidWallSegment() / R_ClipPassWallSegment()** — Subtle clipping logic; easy to introduce overdraw or missing geometry
3. **P_LoadNodes()** — If conversion logic changes, entire BSP tree is corrupted
4. **Angle-to-screen mapping** — Errors cause geometric distortion or clipping bugs

### Medium-Risk Changes
1. **R_RenderBSPNode()** traversal order — Could invert visibility
2. **Segment classification logic** — Could cause walls to disappear or appear incorrectly
3. **Blockmap iteration** — Could cause collision bugs

### Low-Risk Changes
1. **R_FindPlane() caching strategy** — Could slow rendering but not break it
2. **Sprite sorting** — Could cause visual glitches but not crashes
3. **Debug output** — No functional impact if kept behind debug flags

---

## Recommendations for Future Investigation

1. **Formal Testing**: Create test harness for BSP functions (R_PointOnSide, R_RenderBSPNode, etc.)
2. **BSP Tree Analysis**: Profile tree depth, node count, and balance for standard DOOM maps
3. **Performance Profiling**: Measure rendering time by map and subsector count
4. **Numeric Stability**: Check for fixed-point overflow in pathological cases
5. **Sound Integration**: Analyze how BSP interacts with sound system
6. **Multiplayer Verification**: Test BSP consistency in multiplayer scenarios
7. **Automap Verification**: Ensure automap correctly reflects BSP traversal
8. **Edge Case Fuzzing**: Generate intentionally malformed WAD files to test robustness

---

## Summary

This codebase is the original DOOM engine and is well-tested through decades of play. However, formal documentation is sparse, and some behaviors are implicit or undocumented. The main areas of uncertainty are:

- **BSP builder behavior** (not included in source)
- **Numeric precision limits** (not validated)
- **Worst-case performance** (not analyzed)
- **Edge case handling** (not explicitly tested)

These are not defects per se, but rather gaps in documentation that could become issues if the code is modified or ported to new platforms.
