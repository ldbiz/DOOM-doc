# Practical BSP change map

## If I need to change map/WAD lump loading

- **Look first:** `linuxdoom-1.10/doomdata.h`, `w_wad.c`, `w_wad.h`, `p_setup.c`.
- **Why:** `doomdata.h` is the packed format contract; `w_wad.c` supplies lump identity/bytes; `p_setup.c` controls dependency order, conversion, allocation, and cross-linking.
- **Caveats:** Map lumps are addressed by fixed offsets after a marker. Later WADs can override earlier lumps. Existing loaders assume classic short-based records and valid indexes.

## If I need to change BSP node/subsector/seg representation

- **Look first:** `linuxdoom-1.10/doomdata.h`, `r_defs.h`, `p_setup.c`, `r_state.h`, `p_local.h`.
- **Why:** Persistent and runtime layouts are separate; setup converts between them and many modules consume global arrays directly.
- **Caveats:** Nodes keep encoded children while segs become pointer-rich. Subsector sector assignment is delayed until `P_GroupLines`. Renderer and sight code depend on `NF_SUBSECTOR` and root-last conventions.

## If I need to change traversal order

- **Look first:** `linuxdoom-1.10/r_bsp.c` for rendering; `r_main.c` for side classification/point descent; `p_sight.c` for sight traversal.
- **Why:** These are three related but distinct traversals: front-to-back rendering, single-path point location, and trace-crossing sight traversal.
- **Caveats:** Rendering order is coupled to the solid screen-column clip list and back-subtree pruning. Changing side semantics must remain consistent with node-builder conventions.

## If I need to change wall rendering from BSP traversal

- **Look first:** `linuxdoom-1.10/r_bsp.c`, `r_segs.c`, `r_defs.h`.
- **Also inspect:** `r_plane.c` and `r_things.c` for downstream plane/sprite effects.
- **Why:** `R_Subsector` submits segs; `R_AddLine` classifies/clips them; `R_StoreWallRange` draws and creates drawsegs.
- **Caveats:** One-sided, closed, window, empty-trigger, and masked-texture cases affect clipping differently. Visible linedefs are marked for automap display.

## If I need to change visibility or clipping behavior

- **Look first:** `linuxdoom-1.10/r_bsp.c`, `r_segs.c`, `r_plane.c`, `r_things.c`.
- **For gameplay sight:** `linuxdoom-1.10/p_sight.c` and `p_setup.c`'s `REJECT` loading.
- **Why:** Renderer visibility combines view-angle clipping, solid clip ranges, child bounding boxes, drawseg silhouettes, and visplanes. Gameplay sight is a separate REJECT-plus-BSP trace.
- **Caveats:** Renderer visibility and AI line of sight do not share the same traversal or clipping representation.

## If I need to change collision or spatial queries

- **Look first:** `linuxdoom-1.10/p_maputl.c`, `p_map.c`, `p_local.h`, and `p_setup.c`'s `P_LoadBlockMap`.
- **For point-to-sector behavior:** also inspect `r_main.c`'s `R_PointInSubsector` and its callers.
- **Why:** Broad-phase collision/path traversal is BLOCKMAP-driven; BSP contributes point location and line-of-sight traversal.
- **Caveats:** Do not assume a rendering-BSP change will alter movement collision. Things are linked into both sector lists and BLOCKMAP chains.

## If I need to change REJECT or line-of-sight behavior

- **Look first:** `linuxdoom-1.10/p_sight.c`, `p_setup.c`, `doomdata.h`, `r_defs.h`.
- **Why:** `P_CheckSight` performs the sector-pair early-out and initializes the trace; recursive node/leaf functions perform geometric checks.
- **Caveats:** Sight walks subsector segs but deduplicates and tests their parent linedefs. Vertical sector openings narrow slopes along the trace.

## If I need to add BSP debug visualization

- **Look first:** `linuxdoom-1.10/am_map.c` for existing 2D map drawing patterns; `r_bsp.c` and `r_main.c` for traversal/side data; `p_setup.c` for loaded arrays.
- **Why:** The automap is the nearest existing map-space visualization surface, while traversal modules expose the data worth instrumenting.
- **Caveats:** No current BSP debug mode was found. `-devparm` and `-debugfile` are not connected to BSP output, and application-code changes would be required.

## If I need to add or update BSP tests/fixtures

- **Look first:** repository root and `linuxdoom-1.10/Makefile` to establish a test harness; then target `p_setup.c`, `r_bsp.c`, `r_main.c`, and `p_sight.c`.
- **Why:** No automated tests or committed WAD fixtures currently exist.
- **Caveats:** Use legally distributable, minimal WAD fixtures. Isolate global renderer/play state and document expected node-builder conventions explicitly.

## If I need to support another node/map format

- **Look first:** `doomdata.h` and all loaders in `p_setup.c`, then audit `r_bsp.c`, `r_main.c`, and `p_sight.c` assumptions.
- **Why:** The current pipeline is tightly coupled to classic Doom map-lump ordering, 16-bit child encoding, and short-based records.
- **Caveats:** Wider indexes or alternate node encodings affect persistent structures, loader counts/conversion, child-marker logic, global array types, and potentially BLOCKMAP handling.
