# BSP behaviour walkthroughs

## 1. Load a map's prebuilt BSP

| Aspect | Detail |
|---|---|
| Trigger | Starting, warping to, reloading, or transitioning into a level. |
| Files/modules | `d_main.c`, `g_game.c`, `w_wad.c`, `doomdata.h`, `p_setup.c`, `r_defs.h`. |
| Inputs | Selected IWAD/PWAD set; `E#M#` or `MAP##` marker; ordered map lumps. |
| Outputs | Global level arrays for vertices, lines, sides, sectors, segs, subsectors, nodes; cached `REJECT` and `BLOCKMAP`; cross-linked subsector/sector data. |
| Side effects | Old level-tagged zone memory is freed; development WAD may reload; things and specials are spawned after geometry setup. |
| Tests | No automated tests or bundled map fixtures were found. |

**Bird's-eye flow**

`G_DoLoadLevel` calls `P_SetupLevel`, which finds the map marker and loads fixed-offset lumps in dependency order. Packed short-based structures are converted into fixed-point runtime arrays. Seg indexes are resolved to geometry and sector pointers, while `P_GroupLines` assigns each subsector a sector and builds sector line/block bounds.

This code validates little beyond optional compile-time range checks. It expects a valid classic Doom-format, already node-built map.

## 2. Traverse the BSP to render a frame

| Aspect | Detail |
|---|---|
| Trigger | The display loop asks `R_RenderPlayerView` to draw the current player view. |
| Files/modules | `r_main.c`, `r_bsp.c`, `r_segs.c`, `r_plane.c`, `r_things.c`, `r_defs.h`. |
| Inputs | Camera position/angle, loaded nodes/subsectors/segs/sectors, current screen clip state. |
| Outputs | Drawn opaque walls; accumulated drawsegs, visplanes, visible sprites, and masked wall data for later passes. |
| Side effects | Visible linedefs receive `ML_MAPPED`; renderer-global clip and accumulation buffers are updated. |
| Tests | No automated rendering/BSP traversal tests were found. |

**Bird's-eye flow**

After clearing per-frame buffers, rendering begins at the last node. Each node chooses the camera side and recursively processes that child first. The opposite child's BSP bounding box is projected and compared with the horizontal solid-wall clip list; fully hidden subtrees are skipped.

A reached subsector contributes its sector's floor/ceiling planes and thing list, then sends its consecutive segs through view-frustum, backface, and solid/pass-wall clipping. Visible wall ranges are rendered and retained as drawsegs where later sprite or masked-wall clipping needs them.

## 3. Locate a thing or point in the BSP

| Aspect | Detail |
|---|---|
| Trigger | Spawning, moving, relinking, or querying a thing; teleport/deathmatch placement and similar sector-height lookups. |
| Files/modules | `r_main.c`, `p_maputl.c`, `p_map.c`, `p_mobj.c`, `g_game.c`. |
| Inputs | Fixed-point `(x, y)` and loaded BSP nodes/subsectors. |
| Outputs | A `subsector_t*`, normally followed by access to its `sector_t*`. |
| Side effects | `P_SetThingPosition` links the thing into the sector thing list and separately into a BLOCKMAP cell. |
| Tests | No direct unit tests were found. |

**Bird's-eye flow**

`R_PointInSubsector` starts at the root and repeatedly selects a node child according to the point's side of the partition. Once the child marker identifies a subsector, gameplay uses that leaf's assigned sector for floor/ceiling context and sector-based thing lists.

This is the BSP's main collision-adjacent role. Candidate wall and thing collision checks themselves are gathered through the BLOCKMAP grid.

## 4. Check line of sight through BSP leaves

| Aspect | Detail |
|---|---|
| Trigger | Enemy targeting/awareness and gameplay effects that call `P_CheckSight`. |
| Files/modules | `p_sight.c`, `p_setup.c`, `r_defs.h`, `p_enemy.c`. |
| Inputs | Two mobjs, their subsector sectors, `REJECT`, BSP nodes/subsectors/segs, sector heights. |
| Outputs | Boolean unobstructed/blocked sight result. |
| Side effects | Updates `validcount`, per-line visit stamps, and sight counters. |
| Tests | No automated sight/BSP tests were found. |

**Bird's-eye flow**

The two objects' subsector sectors index the sector-pair `REJECT` matrix. A set bit immediately rules out visibility. Otherwise, a sight trace descends the BSP from the root, visiting the start side and only crossing the opposite side when the trace crosses the partition.

At leaves, the trace tests unique linedefs referenced by segs. One-sided lines and closed openings block sight. Two-sided openings narrow top and bottom sight slopes until the target remains visible or the opening closes.

## 5. Perform collision and path traces beside the BSP

| Aspect | Detail |
|---|---|
| Trigger | Movement checks, hitscan attacks, use actions, and other play traces. |
| Files/modules | `p_setup.c`, `p_maputl.c`, `p_map.c`, `p_local.h`. |
| Inputs | Trace or movement bounds, BLOCKMAP origin/grid/lists, runtime linedefs and things. |
| Outputs | Candidate lines/things and ordered intercepts used by the requested play behavior. |
| Side effects | Visit stamps and intercept buffers are updated; movement may relink things. |
| Tests | No automated collision/BLOCKMAP tests were found. |

**Bird's-eye flow**

The BLOCKMAP lump is loaded beside the BSP. `P_PathTraverse` steps through grid cells and asks block iterators for lines and things rather than descending BSP nodes. Movement first uses `R_PointInSubsector` for destination sector heights, then checks nearby BLOCKMAP candidates.

This separation is important when changing spatial behavior: BSP traversal changes affect rendering, point location, and sight; broad collision/query changes usually belong in the BLOCKMAP path.
