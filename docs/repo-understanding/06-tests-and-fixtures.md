# BSP tests and fixtures

## Test framework and commands

No automated test framework, test target, or test source directory was found. The only discoverable build path is the legacy `linuxdoom-1.10/Makefile`, whose default target builds the executable.

Relevant practical commands are therefore:

```sh
make -C linuxdoom-1.10
```

If a compatible IWAD is available, manual BSP exercise can use map-selection arguments such as:

```sh
DOOMWADDIR=/path/to/wads linuxdoom-1.10/linux/linuxxdoom -warp 1 1
DOOMWADDIR=/path/to/wads linuxdoom-1.10/linux/linuxxdoom -file custom.wad -warp 1 1
```

The exact `-warp` form depends on detected game mode: episode/map for Doom-style IWADs and one map number for commercial Doom II-style IWADs.

## Fixtures and sample maps

No `.wad` or `.lmp` fixture files are committed. The source expects external copyrighted/game or user-created WAD data. Consequently, the repository does not provide a stable known node tree, subsector set, REJECT matrix, or BLOCKMAP against which behavior can be asserted.

The `-wart` development flow refers to external development-map paths, but those maps are not present.

## What the code itself documents

In the absence of tests, these implementation paths act as the strongest executable documentation:

| Behavior | Code path that documents it |
|---|---|
| Packed-to-runtime BSP loading | `P_LoadNodes`, `P_LoadSubsectors`, `P_LoadSegs`, and `P_GroupLines` in `p_setup.c`. |
| Root/leaf conventions | `R_RenderBSPNode`, `R_PointInSubsector`, and `P_CrossBSPNode`. |
| Front-to-back rendering and child-box pruning | `R_RenderBSPNode` and `R_CheckBBox` in `r_bsp.c`. |
| Leaf-to-render-products conversion | `R_Subsector`, `R_AddLine`, `R_StoreWallRange`, and plane/sprite modules. |
| Point-to-sector gameplay lookup | `R_PointInSubsector` callers, especially `P_SetThingPosition` and movement checks. |
| Sight traversal | `P_CheckSight`, `P_CrossBSPNode`, and `P_CrossSubsector` in `p_sight.c`. |
| Collision's separation from BSP | `P_BlockLinesIterator` and `P_PathTraverse` in `p_maputl.c`. |

## Obvious BSP coverage gaps

- No loader tests for malformed lengths, invalid indexes, bad child markers, empty subsectors, or endian conversion.
- No deterministic traversal-order or bounding-box-pruning tests.
- No visual regression tests for walls, visplanes, sprites, clipping, or masked textures.
- No tests comparing `R_PointInSubsector` results with known points/leaves.
- No sight tests for one-sided walls, windows, doors, REJECT early-outs, or traces lying on partition lines.
- No tests documenting single-subsector maps or special `-1` child handling.
- No tests defining whether limits such as `MAXDRAWSEGS` and `MAXVISPLANES` fail acceptably.
- No fixture demonstrating the intended relationship among BSP, REJECT, and BLOCKMAP data.

## Useful future fixture shapes

Without prescribing implementation changes, a focused fixture set would ideally include:

- A one-subsector map with no nodes.
- A two-leaf map with an obvious partition and known front/back points.
- A two-sided opening whose floor/ceiling differences exercise sight slopes and wall clipping.
- A map where a back subtree bounding box becomes fully occluded after rendering the front child.
- A map with a deliberately populated REJECT entry and a BLOCKMAP path crossing several cells.
