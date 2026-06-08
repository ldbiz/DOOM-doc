# BSP-Related Behaviour Walkthroughs

## Behaviour 1: Loading a Map and Initializing the BSP Tree

### User/System Action
Game loads a map level (e.g., E1M1).

### Files Involved
- `g_game.c` — High-level game logic calls `P_SetupLevel()`
- `p_setup.c` — All map loading functions
- `w_wad.c` — WAD I/O and lump caching
- `doomstat.c` / `doomstat.h` — Global arrays for map data
- `z_zone.c` — Memory allocation

### Bird's-Eye Flow

```
G_InitNew(skill, episode, map)
  ↓
P_SetupLevel(map)
  ├─ For each lump type in order:
  │  ├─ W_LumpLength() — Determine count
  │  ├─ Z_Malloc() — Allocate runtime array
  │  ├─ W_CacheLumpNum() — Load lump data
  │  ├─ Iterate WAD structures, convert to runtime format
  │  └─ Z_Free() — Release cache
  │
  ├─ Load VERTEXES → vertexes[] array
  ├─ Load LINEDEFS → lines[] array
  ├─ Load SIDEDEFS → sides[] array
  ├─ Load SECTORS → sectors[] array
  ├─ Load SEGS → segs[] array (pointers wired to lines, sides, sectors, vertices)
  ├─ Load SSECTORS → subsectors[] array (pointers wired to sectors, segs)
  ├─ Load NODES → nodes[] array (BSP tree root = numnodes - 1)
  ├─ Load BLOCKMAP → blockmap[] array (spatial grid)
  └─ Load REJECT → rejectmatrix (sight culling table)

Ready for gameplay
```

### Inputs
- Map lump index (or name)
- WAD file containing the map (already opened)

### Outputs
- Global arrays: `vertexes`, `lines`, `sides`, `sectors`, `segs`, `subsectors`, `nodes`
- Global spatial structures: `blockmap`, `rejectmatrix`
- All cross-references wired (pointers established between structures)

### External Calls / Side Effects
- Allocates all BSP and map memory from the zone allocator
- Initializes thing spawning (P_SpawnMapThing for each THINGS lump entry)
- Clears previous level data
- Sets up initial player position and angle

### Tests Covering This
No explicit unit tests in this codebase (it's classic id Software code without a test framework). Functional correctness is verified by running the game and checking that maps load and render correctly.

---

## Behaviour 2: Traversing the BSP Tree for Rendering a Frame

### User/System Action
Player is in-game; rendering a new frame from player's current position/angle.

### Files Involved
- `r_main.c` — View setup, point-on-side testing
- `r_bsp.c` — BSP traversal (`R_RenderBSPNode`, `R_Subsector`)
- `r_segs.c` — Segment clipping and registration
- `r_plane.c` — Plane caching for floors/ceilings
- `r_things.c` — Sprite registration

### Bird's-Eye Flow

```
R_RenderPlayerView()
  ├─ R_SetupFrame(player)
  │  └─ Set viewx, viewy, viewz, viewangle (from player mobj)
  │
  ├─ R_ClearClipSegs()
  │  └─ Initialize solidsegs[] (entire screen is open)
  │
  ├─ R_ClearDrawSegs()
  │  └─ Reset drawsegs[] pointer
  │
  ├─ R_RenderBSPNode(numnodes - 1)
  │  │
  │  ├─ Recursive traversal:
  │  │  ├─ If subsector:
  │  │  │  └─ R_Subsector(num)
  │  │  │     ├─ Find planes for sector
  │  │  │     ├─ R_AddSprites(sector)
  │  │  │     └─ For each seg in subsector:
  │  │  │        └─ R_AddLine(seg)
  │  │  │
  │  │  ├─ Else (interior node):
  │  │  │  ├─ R_PointOnSide(viewx, viewy, node)
  │  │  │  │  └─ Determine front/back
  │  │  │  │
  │  │  │  ├─ R_RenderBSPNode(children[front])
  │  │  │  │  └─ Recursive call for front
  │  │  │  │
  │  │  │  └─ R_CheckBBox(node->bbox[back])
  │  │  │     └─ If visible:
  │  │  │        └─ R_RenderBSPNode(children[back])
  │
  ├─ R_DrawPlanes()
  │  └─ Draw all cached floor/ceiling visplanes
  │
  ├─ R_DrawMaskedSegs()
  │  └─ Draw two-sided wall textures
  │
  └─ R_DrawSprites()
     └─ Draw all sprites collected during traversal

Frame complete
```

### Inputs
- Player position and angle (from player mobj in current sector)
- BSP tree (nodes[], subsectors[], segs[])
- Map geometry (sectors[], lines[], sides[], vertices[])

### Outputs
- Screen filled with rendered frame (pixel data)
- Internal data structures updated: solidsegs[], drawsegs[], visplanes[], vissprites[]

### External Calls / Side Effects
- Modifies global state: viewx, viewy, viewz, viewangle, frontsector, backsector
- Builds solidsegs[] and drawsegs[] arrays
- Caches visplanes for the frame
- Collects sprite records for sorting

### Tests Covering This
No explicit tests; functionality is verified by gameplay. Visually, correct BSP traversal results in correct back-to-front rendering with no clipping artifacts.

---

## Behaviour 3: Wall Segment Clipping and Classification

### User/System Action
During frame rendering, a segment is encountered and must be classified (solid, pass-through, or invisible) and clipped to visible screen area.

### Files Involved
- `r_bsp.c` — R_AddLine() entry point
- `r_segs.c` — Clipping logic, R_ClipSolidWallSegment(), R_ClipPassWallSegment()
- `r_main.c` — Angle/coordinate conversion utilities

### Bird's-Eye Flow

```
R_AddLine(seg)
  ├─ Convert segment endpoints to view angles
  │  └─ R_PointToAngle() for each vertex
  │
  ├─ Backface cull check
  │  └─ If angle span >= 180°, segment faces away → return
  │
  ├─ Clip to view frustum
  │  └─ Compute screen x coordinates (x1, x2)
  │
  ├─ Classify wall type:
  │
  ├─ If one-sided linedef:
  │  └─ Mark as solid (opaque wall)
  │
  ├─ Else if back sector fully blocks front sector:
  │  └─ Mark as solid (closed door)
  │
  ├─ Else if sectors differ in height/texture:
  │  └─ Mark as pass-through (window/partial wall)
  │
  ├─ Else (identical sectors):
  │  └─ Return (invisible line)
  │
  ├─ Clip to screen coverage:
  │  ├─ If solid:
  │  │  └─ R_ClipSolidWallSegment(x1, x2)
  │  │     ├─ Check against solidsegs[]
  │  │     ├─ For visible parts: R_StoreWallRange()
  │  │     └─ Update solidsegs[] to mark blocked columns
  │  │
  │  └─ If pass-through:
  │     └─ R_ClipPassWallSegment(x1, x2)
  │        ├─ Check against solidsegs[] for reference only
  │        └─ For visible parts: R_StoreWallRange()
  │           (does NOT update solidsegs[])
  │
  └─ Return

Later: R_StoreWallRange() records drawseg_t entry for rendering
```

### Inputs
- seg_t structure (vertices, linedef, sidedef, sectors)
- Current viewpoint (viewx, viewy, viewangle)
- Current clip list (solidsegs[])

### Outputs
- Updated solidsegs[] (if segment is solid)
- drawseg_t entry added to drawsegs[] array
- Global state updated: curline, sidedef, linedef, frontsector, backsector

### External Calls / Side Effects
- May call R_StoreWallRange() multiple times (for non-contiguous visible pieces)
- Modifies solidsegs[] by inserting/merging/removing entries
- Updates clip list; later segments may be skipped if already fully blocked

### Tests Covering This
No explicit tests. Correct clipping prevents overdraw, visual artifacts, or rendering errors.

---

## Behaviour 4: Point-on-Side Testing (BSP Partition Query)

### User/System Action
During BSP traversal or spatial query, need to determine which side of a partition line a point lies on.

### Files Involved
- `r_main.c` — R_PointOnSide(), R_PointOnSegSide()
- `r_bsp.c` — Called during R_RenderBSPNode()
- `p_map.c` — May be used for collision testing
- `m_fixed.h` — Fixed-point multiplication

### Bird's-Eye Flow

```
R_PointOnSide(x, y, node)
  ├─ Extract partition line from node
  │  ├─ node->x, node->y (point)
  │  └─ node->dx, node->dy (direction vector)
  │
  ├─ If partition is vertical (dx == 0):
  │  └─ Compare x against node->x; return based on dy sign
  │
  ├─ Else if partition is horizontal (dy == 0):
  │  └─ Compare y against node->y; return based on dx sign
  │
  ├─ Else (general case):
  │  ├─ Compute cross product: (x - node->x) * dy - (y - node->y) * dx
  │  │  └─ Uses fixed-point FixedMul()
  │  │
  │  ├─ If result < 0: front side (return 0)
  │  └─ If result >= 0: back side (return 1)
  │
  └─ Return side (0 or 1)

Result used to decide traversal direction in BSP tree
```

### Inputs
- Point coordinates (x, y) in world space
- BSP node with partition line parameters

### Outputs
- Side index (0 = front, 1 = back)

### External Calls / Side Effects
- Pure calculation, no side effects
- Used to drive control flow in BSP traversal

### Tests Covering This
No explicit tests. Correctness verified by ensuring BSP traversal produces correct rendering order.

---

## Behaviour 5: Collision Detection with Blockmap

### User/System Action
Player/object moves or needs to test for collision with walls/objects.

### Files Involved
- `p_map.c` — P_CheckPosition(), P_PathTraverse()
- `p_maputl.c` — Blockmap iteration utilities
- `p_local.h` — MAPBLOCK* constants
- `doomstat.c` — blockmap[], blockmaplump, blocklinks[]

### Bird's-Eye Flow

```
P_CheckPosition(thing, x, y)
  ├─ Compute bounding box of moving thing
  │  └─ bbox = [x - radius, x + radius, y - radius, y + radius]
  │
  ├─ Convert bbox to blockmap coordinates
  │  └─ Iterate over blocks containing the bbox
  │
  ├─ For each block in blockmap:
  │  ├─ Get list of things in block (via blocklinks[])
  │  │  └─ P_ThingBlockmapInteract()
  │  │
  │  ├─ Get list of lines in block (via blockmap lump)
  │  │  └─ P_LineBlockmapInteract()
  │  │
  │  └─ Test collision with each object/line
  │     └─ If collision: return false (can't move)
  │
  └─ If no collision: return true (can move)

Result determines whether thing moves or stays in place
```

### Inputs
- Thing (mobj_t) with position and radius
- Target position (x, y)
- Blockmap acceleration structure

### Outputs
- true/false (can move or not)
- May set tmfloorz, tmceilingz (height constraints)

### External Calls / Side Effects
- May call line-of-sight or segment-crossing tests
- May modify thing's position if movement is valid
- No modification to BSP tree

### Tests Covering This
No explicit tests. Correctness verified by gameplay: collisions should be solid; no clipping through walls.

---

## Summary Table

| Behaviour | Entry | Key Function | Purpose | Output |
|-----------|-------|--------------|---------|--------|
| 1. Map Loading | P_SetupLevel | P_LoadVertexes, P_LoadNodes, P_LoadSegs | Initialize BSP and map | Global arrays wired |
| 2. Frame Render | R_RenderPlayerView | R_RenderBSPNode | Traverse BSP, collect visible geometry | Screen-space data |
| 3. Segment Clipping | R_AddLine | R_ClipSolidWallSegment | Classify and clip walls | drawsegs[], solidsegs[] |
| 4. Point-on-Side | R_RenderBSPNode | R_PointOnSide | Test partition membership | Side index |
| 5. Collision Check | P_Move | P_CheckPosition | Spatial query for collision | Move allowed? |
