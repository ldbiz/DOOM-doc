# 06 - BSP Tests & Fixtures

## Test framework

This codebase has **no automated test framework, no unit tests, and no test fixtures**. The linuxdoom-1.10 release is the raw source code from id Software (1997), predating common unit testing practices in game development.

The `Makefile` in `linuxdoom-1.10/` builds a single binary (`linux/doom`) with no test targets.

## How to exercise BSP behaviour

BSP behaviour can only be verified by:

1. **Running the full application** with a valid DOOM IWAD (e.g., `doom1.wad`, `doom2.wad`). The game must be launched with `-iwad <path>`.
2. **Observing rendering output** — visual verification that walls, floors, ceilings, and sprites render correctly through the BSP traversal.
3. **Observing monster AI** — monsters should correctly detect or fail to detect the player based on `P_CheckSight()` BSP traversal.
4. **Using debugging tools** — breakpoints in `R_RenderBSPNode()`, `R_Subsector()`, `R_AddLine()`, `P_CheckSight()`, or `P_CrossBSPNode()` to inspect BSP traversal state.

## BSP coverage gaps

The following BSP behaviours have **no explicit verification** in this codebase:

| Behaviour | Gap |
|---|---|
| Correct node loading from WAD | No validation that `numnodes`, `numsubsectors`, `numsegs` are consistent. If the WAD is malformed, behaviour is undefined. |
| BSP traversal correctness | No reference rendering to compare against. Subtle errors (e.g., wrong partition side test) would only manifest as visual glitches. |
| Visplane merging correctness | `R_FindPlane()` and `R_CheckPlane()` logic is complex; no unit tests verify correct plane reuse vs. splitting. |
| Solidseg clipping correctness | `R_ClipSolidWallSegment()` and `R_ClipPassWallSegment()` are manually verified only. |
| Sprite clipping against drawsegs | `R_DrawSprite()` clipping math against `drawsegs[]` has no automated tests. |
| REJECT table correctness | No validation that `rejectmatrix` size matches `numsectors * numsectors / 8`. |
| Overflow handling | `MAXDRAWSEGS`, `MAXVISPLANES`, `MAXVISSPRITES` overflow triggers `I_Error()` — no graceful degradation. |

## Known test data

- **No test WADs included** — The repo contains no WAD files. A commercial DOOM IWAD is required.
- **No sample maps** — No smaller/focused test maps exist in this repo.
- **No BSP validation tools** — No tools to dump or verify BSP trees.

## What the code does verify at runtime

Despite lacking tests, the code includes some runtime safety checks:

- `R_DrawPlanes()` checks `lastvisplane - visplanes == MAXVISPLANES` and calls `I_Error("R_FindPlane: no more visplanes")`.
- `R_StoreWallRange()` checks `ds_p - drawsegs == MAXDRAWSEGS` and calls `I_Error("R_StoreWallRange: drawsegs overflow")`.
- `R_NewVisSprite()` checks `vissprite_p - vissprites == MAXVISSPRITES` and calls `I_Error("R_NewVisSprite: vissprites overflow")`.

These serve as operational guards but are not systematic tests.

## Recommendations for adding BSP tests

If tests were to be added (not in scope for this documentation), key areas to cover:

1. **BSP loading**: Parse known WADs and verify `numnodes`, `numsubsectors`, `numsegs` match expected values; verify coordinate scaling.
2. **Point-in-subsector**: For a known map, verify that specific world coordinates resolve to the expected subsector index.
3. **Partition side test**: Verify `R_PointOnSide()` returns correct results for points on known sides of partition lines.
4. **Traversal order**: Verify that `R_RenderBSPNode()` visits subsectors in the correct front-to-back order for a given viewpoint.
5. **LOS correctness**: Verify `P_CheckSight()` returns expected results for known-visible and known-occluded object pairs.
