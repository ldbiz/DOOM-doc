# BSP concepts in this repository

## Map marker and map lumps

- **Doom/BSP meaning:** A map is a marker such as `E1M1` or `MAP01` followed by an ordered group of geometry and spatial-index lumps.
- **In this repo:** `P_SetupLevel` resolves the marker and loads fields by `lumpnum + ML_*`; ordering is therefore part of the format contract, not discovered by individual lump name.
- **Appears in:** `ML_*` enum, `P_SetupLevel`, WAD directory lookup/cache functions.
- **Related files:** `doomdata.h`, `p_setup.c`, `w_wad.c`, `d_main.c`.
- **Relations:** `VERTEXES`, `LINEDEFS`, `SIDEDEFS`, and `SECTORS` describe map geometry; `SEGS`, `SSECTORS`, and `NODES` are the prebuilt BSP; `REJECT` and `BLOCKMAP` are adjacent acceleration data.

## BSP node and partition line

- **Doom/BSP meaning:** An internal tree node divides 2D space with a directed partition line and points to two child regions.
- **In this repo:** `mapnode_t` stores 16-bit `(x, y, dx, dy)` values; `P_LoadNodes` expands them to 16.16 fixed-point `node_t` values. The root is assumed to be `nodes[numnodes - 1]`.
- **Appears in:** `mapnode_t`, `node_t`, `P_LoadNodes`, `R_RenderBSPNode`, `R_PointInSubsector`, `P_CrossBSPNode`.
- **Related files:** `doomdata.h`, `r_defs.h`, `p_setup.c`, `r_bsp.c`, `r_main.c`, `p_sight.c`.
- **Relations:** Side tests select a child; children resolve to more nodes or subsectors. Renderer traversal also uses each child's bounding box.

## Front/back child and `NF_SUBSECTOR`

- **Doom/BSP meaning:** Node children identify either another internal node or a leaf subsector; side 0 is front and side 1 is back relative to the directed partition.
- **In this repo:** Child identifiers are 16-bit values. Bit `0x8000` (`NF_SUBSECTOR`) marks a subsector index. Rendering visits the viewer-side child first; sight traversal visits the trace-start side first.
- **Appears in:** `mapnode_t.children`, `node_t.children`, `R_PointOnSide`, `R_RenderBSPNode`, `P_DivlineSide`, `P_CrossBSPNode`.
- **Related files:** `doomdata.h`, `r_defs.h`, `r_main.c`, `r_bsp.c`, `p_sight.c`.
- **Relations:** Clearing the marker yields the subsector index. A special `-1` child is treated as subsector zero by rendering and sight code.

## Child bounding box

- **Doom/BSP meaning:** Each node stores an axis-aligned bound for each child subtree so a renderer can reject regions without descending into them.
- **In this repo:** `bbox[2][4]` is fixed-point at runtime. `R_CheckBBox` projects the relevant silhouette corners and checks the current solid screen-column clip list.
- **Appears in:** `mapnode_t.bbox`, `node_t.bbox`, `R_CheckBBox`, `R_RenderBSPNode`.
- **Related files:** `doomdata.h`, `r_defs.h`, `p_setup.c`, `r_bsp.c`, `m_bbox.h`.
- **Relations:** It prunes the back child after front-to-back rendering has potentially added occluding wall spans. It is not the BLOCKMAP and is not used by sight traversal.

## Subsector

- **Doom/BSP meaning:** A convex BSP leaf represented by a consecutive list of segs; it lies within one sector.
- **In this repo:** `subsector_t` stores `firstline`, `numlines`, and a runtime sector pointer. `P_GroupLines` assigns the sector from the first seg's sidedef sector.
- **Appears in:** `mapsubsector_t`, `subsector_t`, `P_LoadSubsectors`, `P_GroupLines`, `R_Subsector`, `R_PointInSubsector`, `P_CrossSubsector`.
- **Related files:** `doomdata.h`, `r_defs.h`, `p_setup.c`, `r_bsp.c`, `r_main.c`, `p_sight.c`.
- **Relations:** Leaves group segs; their sector supplies floor, ceiling, lighting, and thing lists.

## Seg

- **Doom/BSP meaning:** A directed line segment produced when a BSP builder splits linedefs along partition boundaries.
- **In this repo:** `P_LoadSegs` resolves vertex, linedef, sidedef, front-sector, and optional back-sector pointers. Rendering iterates segs in a reached subsector; line of sight uses their parent linedefs and sector openings.
- **Appears in:** `mapseg_t`, `seg_t`, `P_LoadSegs`, `R_AddLine`, `R_StoreWallRange`, `P_CrossSubsector`.
- **Related files:** `doomdata.h`, `r_defs.h`, `p_setup.c`, `r_bsp.c`, `r_segs.c`, `p_sight.c`.
- **Relations:** A seg belongs to a subsector but refers back to editor geometry and visual sector sides.

## Linedef, sidedef, and sector

- **Doom/BSP meaning:** A linedef is original map geometry; a sidedef supplies textures and a facing sector; a sector supplies floor/ceiling heights, flats, light, and behavior.
- **In this repo:** Loaders create pointer-linked runtime records. Segs select one linedef side, establishing front/back sectors used for wall classification and sight openings.
- **Appears in:** `maplinedef_t`, `mapsidedef_t`, `mapsector_t`, runtime `line_t`, `side_t`, `sector_t`, their loader functions.
- **Related files:** `doomdata.h`, `r_defs.h`, `p_setup.c`, `r_bsp.c`, `r_segs.c`, `p_sight.c`.
- **Relations:** Nodes organize space; segs expose portions of linedefs; sides connect those portions to sectors; sectors determine planes and vertical openings.

## Vertex and fixed-point coordinates

- **Doom/BSP meaning:** Vertices anchor map lines and segs in 2D map space.
- **In this repo:** On-disk signed shorts are byte-swapped as needed and shifted by `FRACBITS` into signed 16.16 `fixed_t` coordinates. Nodes, boxes, heights, and offsets undergo similar conversion.
- **Appears in:** `mapvertex_t`, `vertex_t`, `P_LoadVertexes`, `P_LoadNodes`, `m_fixed.h`.
- **Related files:** `doomdata.h`, `r_defs.h`, `p_setup.c`, `m_fixed.h`, `m_swap.h`.
- **Relations:** Side tests, projections, collisions, and sector heights share the fixed-point coordinate model.

## Point-to-subsector lookup

- **Doom/BSP meaning:** Descend partition nodes according to which side contains a point until reaching a leaf.
- **In this repo:** `R_PointInSubsector` iteratively starts at the last node and uses `R_PointOnSide`. It is renderer-owned by name but widely used by play code.
- **Appears in:** object placement, movement checks, teleport/deathmatch effects, and other sector-height queries.
- **Related files:** `r_main.c`, `p_maputl.c`, `p_map.c`, `p_mobj.c`, `g_game.c`.
- **Relations:** The returned subsector provides a sector and supports thing-to-sector linking; BLOCKMAP linking happens separately.

## Solid clipping and drawseg

- **Doom/BSP meaning:** Front-to-back wall rendering can record covered horizontal screen ranges; visible walls also leave metadata for later masked-wall and sprite clipping.
- **In this repo:** `solidsegs` lets `R_CheckBBox` and wall clipping reject occluded ranges. `R_StoreWallRange` emits `drawseg_t` records with scales, silhouettes, and clip pointers.
- **Appears in:** `R_ClearClipSegs`, `R_ClipSolidWallSegment`, `R_ClipPassWallSegment`, `R_CheckBBox`, `R_StoreWallRange`, `R_DrawMasked`.
- **Related files:** `r_bsp.c`, `r_defs.h`, `r_segs.c`, `r_things.c`.
- **Relations:** BSP traversal order makes solid clipping effective; drawsegs preserve wall effects needed after traversal.

## Visplane

- **Doom/BSP meaning:** A screen-space accumulation of visible floor or ceiling regions sharing height, flat, and light.
- **In this repo:** `R_Subsector` selects floor/ceiling visplanes from its sector. Wall rendering marks their column extents, and `R_DrawPlanes` draws them after BSP traversal.
- **Appears in:** `visplane_t`, `R_FindPlane`, `R_CheckPlane`, `R_DrawPlanes`.
- **Related files:** `r_defs.h`, `r_bsp.c`, `r_segs.c`, `r_plane.c`, `r_main.c`.
- **Relations:** Visplanes are products of rendering BSP leaves, not nodes in the BSP itself.

## REJECT

- **Doom/BSP meaning:** A sector-to-sector bit matrix that can rule out visibility without geometric tracing.
- **In this repo:** The raw `REJECT` lump is cached as `rejectmatrix`. `P_CheckSight` indexes it using each object's subsector sector before attempting BSP traversal.
- **Appears in:** `ML_REJECT`, `rejectmatrix`, `P_CheckSight`.
- **Related files:** `doomdata.h`, `p_setup.c`, `p_sight.c`, `p_local.h`.
- **Relations:** It is a coarse early-out adjacent to, but structurally separate from, the BSP.

## BLOCKMAP

- **Doom/BSP meaning:** A uniform map grid listing linedefs per block, used to accelerate movement and trace queries.
- **In this repo:** `P_LoadBlockMap` preserves the lump, exposes its offset table, and allocates dynamic thing chains. Movement, attacks, and use traces iterate grid cells rather than BSP leaves.
- **Appears in:** `ML_BLOCKMAP`, `P_LoadBlockMap`, `P_BlockLinesIterator`, `P_BlockThingsIterator`, `P_PathTraverse`.
- **Related files:** `doomdata.h`, `p_setup.c`, `p_local.h`, `p_maputl.c`, `p_map.c`.
- **Relations:** It complements BSP point lookup and sight traversal but is the principal broad-phase structure for collision/path queries.
