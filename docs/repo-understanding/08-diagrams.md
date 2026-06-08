# Diagrams

## Diagram 1: BSP Data Loading Flow

```mermaid
graph TD
    A["Game Start"] --> B["P_SetupLevel"]
    
    B --> C["W_CacheLumpNum VERTEXES"]
    C --> D["P_LoadVertexes"]
    D --> E["vertexes[] created<br/>16-bit→32-bit conversion"]
    
    B --> F["W_CacheLumpNum LINEDEFS"]
    F --> G["P_LoadLineDefs"]
    G --> H["lines[] created<br/>Vertex pointers wired"]
    
    B --> I["W_CacheLumpNum SIDEDEFS"]
    I --> J["P_LoadSideDefs"]
    J --> K["sides[] created<br/>Sector pointers wired"]
    
    B --> L["W_CacheLumpNum SECTORS"]
    L --> M["P_LoadSectors"]
    M --> N["sectors[] created<br/>Texture/height loaded"]
    
    B --> O["W_CacheLumpNum SEGS"]
    O --> P["P_LoadSegs"]
    P --> Q["segs[] created<br/>Vertices, linedefs,<br/>sidedefs, sectors wired"]
    
    B --> R["W_CacheLumpNum SSECTORS"]
    R --> S["P_LoadSubsectors"]
    S --> T["subsectors[] created<br/>Sector & seg pointers wired"]
    
    B --> U["W_CacheLumpNum NODES"]
    U --> V["P_LoadNodes"]
    V --> W["nodes[] created<br/>BSP tree initialized<br/>Root = nodes[numnodes-1]"]
    
    B --> X["W_CacheLumpNum BLOCKMAP"]
    X --> Y["P_LoadBlockMap"]
    Y --> Z["blockmap[] created<br/>Spatial hash grid ready"]
    
    B --> AA["W_CacheLumpNum REJECT"]
    AA --> AB["P_LoadReject"]
    AB --> AC["rejectmatrix created<br/>Sight culling ready"]
    
    E --> AD["Map Ready"]
    H --> AD
    K --> AD
    N --> AD
    Q --> AD
    T --> AD
    W --> AD
    Z --> AD
    AC --> AD
    
    AD --> AE["P_SpawnMapThing"]
    AE --> AF["Gameplay Starts"]
```

---

## Diagram 2: BSP Runtime Traversal & Rendering Flow

```mermaid
graph TD
    A["R_RenderPlayerView"] --> B["R_SetupFrame<br/>Set viewx, viewy, viewz, viewangle"]
    B --> C["R_ClearClipSegs<br/>Screen is open"]
    C --> D["R_ClearDrawSegs"]
    D --> E["R_RenderBSPNode<br/>numnodes - 1"]
    
    E --> F{"bspnum &<br/>NF_SUBSECTOR?"}
    
    F -->|Yes| G["R_Subsector<br/>Process BSP leaf"]
    G --> H["Find sector<br/>floor/ceiling planes"]
    H --> I["R_FindPlane<br/>Cache visplane"]
    I --> J["R_AddSprites<br/>Register sector sprites"]
    J --> K["For each segment<br/>in subsector:"]
    K --> L["R_AddLine<br/>Process wall segment"]
    
    L --> M["Backface cull?"]
    M -->|Yes| N["Skip"]
    M -->|No| O["Classify wall type"]
    
    O --> P{"Wall type?"}
    P -->|One-sided| Q["Solid"]
    P -->|Closed| Q
    P -->|Window| R["Pass-through"]
    P -->|Empty| S["Skip"]
    
    Q --> T["R_ClipSolidWallSegment<br/>Update solidsegs[]"]
    R --> U["R_ClipPassWallSegment<br/>No clip update"]
    S --> N
    
    T --> V["R_StoreWallRange<br/>Add drawseg"]
    U --> V
    V --> W["Next segment"]
    W --> K
    K --> X["All segments done"]
    X --> Y["Return to parent node"]
    
    F -->|No| Z["Interior node<br/>Get partition line"]
    Z --> AA["R_PointOnSide<br/>viewx, viewy"]
    AA --> AB["side = 0 or 1"]
    AB --> AC["R_RenderBSPNode<br/>children[side]<br/>Visit front"]
    AC --> E
    
    AB --> AD["R_CheckBBox<br/>children[side^1]<br/>Visible?"]
    AD -->|No| AE["Skip back"]
    AD -->|Yes| AF["R_RenderBSPNode<br/>children[side^1]<br/>Visit back"]
    AF --> E
    AE --> AY["Return"]
    
    Y --> AY
    
    AY --> AG["All nodes traversed"]
    AG --> AH["R_DrawPlanes<br/>Floors/ceilings"]
    AH --> AI["R_DrawMaskedSegs<br/>Two-sided walls"]
    AI --> AJ["R_DrawSprites<br/>All sprites"]
    AJ --> AK["Frame rendered"]
```

---

## Diagram 3: BSP Node Structure & Children Relationships

```mermaid
graph TD
    A["node_t: Interior Node"] --> B["Partition line<br/>x, y, dx, dy"]
    A --> C["Bounding box[2]<br/>Front child bbox<br/>Back child bbox"]
    A --> D["children[2]<br/>0: Front child<br/>1: Back child"]
    
    B --> E["Front Space<br/>Toward normal"]
    B --> F["Back Space<br/>Away from normal"]
    
    D --> G["Front Child<br/>children[0]"]
    D --> H["Back Child<br/>children[1]"]
    
    G --> I{"Is subsector?<br/>& NF_SUBSECTOR"}
    I -->|Yes| J["subsector_t<br/>Leaf with segs"]
    I -->|No| K["node_t<br/>Another partition"]
    
    H --> L{"Is subsector?<br/>& NF_SUBSECTOR"}
    L -->|Yes| M["subsector_t<br/>Leaf with segs"]
    L -->|No| N["node_t<br/>Another partition"]
    
    J --> O["sector_t"]
    J --> P["firstline, numlines"]
    P --> Q["segs[] array<br/>Wall segments"]
    Q --> R["Each seg references<br/>linedef, sidedef,<br/>vertices"]
    
    O --> S["floorheight<br/>ceilingheight"]
    O --> T["floorpic<br/>ceilingpic<br/>lightlevel"]
```

---

## Diagram 4: Segment Classification & Clipping

```mermaid
graph TD
    A["seg_t in subsector"] --> B["R_AddLine"]
    
    B --> C["Convert to view angles"]
    C --> D["Backface cull?"]
    D -->|Yes| E["Return"]
    D -->|No| F["Clip to frustum"]
    
    F --> G["Map screen coordinates"]
    G --> H["Determine wall type"]
    
    H --> I{"backsector?"}
    I -->|No| J["One-sided wall"]
    I -->|Yes| K{"Heights same?"}
    
    K -->|No| L["Window wall"]
    K -->|Yes| M{"Textures same<br/>& no middle?"}
    M -->|No| N["Two-sided opaque"]
    M -->|Yes| O["Empty line"]
    
    J --> P["Solid"]
    N --> P
    L --> Q["Pass-through"]
    O --> E
    
    P --> R["R_ClipSolidWallSegment<br/>Updates solidsegs[]"]
    Q --> S["R_ClipPassWallSegment<br/>Doesn't update solidsegs[]"]
    
    R --> T["R_StoreWallRange<br/>drawseg_t added"]
    S --> T
    T --> U["Wall queued for rendering"]
```

---

## Diagram 5: Sequence: Frame Rendering from Viewpoint

```mermaid
sequenceDiagram
    player ->> engine: Frame render request
    engine ->> renderer: R_RenderPlayerView()
    renderer ->> renderer: R_SetupFrame()
    
    renderer ->> bsp_tree: R_RenderBSPNode(root)
    
    loop Recursive traversal
        alt Is subsector?
            bsp_tree ->> subsector: yes → R_Subsector(num)
            subsector ->> sector: Get floor/ceiling
            subsector ->> planes: R_FindPlane()
            subsector ->> sprites: R_AddSprites()
            
            loop Each segment
                subsector ->> segment_proc: R_AddLine(seg)
                segment_proc ->> seg_class: Classify wall
                alt Solid wall
                    seg_class ->> clip: R_ClipSolidWallSegment()
                    clip ->> solidsegs: Update clip list
                else Pass-through
                    seg_class ->> clip: R_ClipPassWallSegment()
                else Empty line
                    seg_class ->> segment_proc: Skip
                end
                segment_proc ->> drawsegs: R_StoreWallRange()
            end
        else Interior node
            bsp_tree ->> partition: R_PointOnSide(view)
            partition ->> bsp_tree: Visit front child
            bsp_tree ->> culling: R_CheckBBox(back)
            alt Visible
                culling ->> bsp_tree: Visit back child
            else Not visible
                culling ->> bsp_tree: Skip back
            end
        end
    end
    
    bsp_tree ->> renderer: Traversal complete
    renderer ->> planes: R_DrawPlanes()
    renderer ->> walls: R_DrawMaskedSegs()
    renderer ->> sprites: R_DrawSprites()
    renderer ->> player: Frame complete
```

---

## Diagram 6: Data Dependency Graph

```mermaid
graph TD
    WAD["WAD File<br/>Binary lumps"]
    
    WAD --> VTX["VERTEXES<br/>mapvertex_t[]"]
    WAD --> LDF["LINEDEFS<br/>maplinedef_t[]"]
    WAD --> SDF["SIDEDEFS<br/>mapsidedef_t[]"]
    WAD --> SEC["SECTORS<br/>mapsector_t[]"]
    WAD --> SEG["SEGS<br/>mapseg_t[]"]
    WAD --> SSC["SSECTORS<br/>mapsubsector_t[]"]
    WAD --> NOD["NODES<br/>mapnode_t[]"]
    
    VTX --> V_RT["vertex_t[]<br/>Runtime"]
    LDF --> L_RT["line_t[]<br/>Runtime"]
    SDF --> SD_RT["side_t[]<br/>Runtime"]
    SEC --> S_RT["sector_t[]<br/>Runtime"]
    
    V_RT --> SEG_PROC["P_LoadSegs<br/>Wire pointers"]
    L_RT --> SEG_PROC
    SD_RT --> SEG_PROC
    S_RT --> SEG_PROC
    
    SEG_PROC --> SEG_RT["seg_t[]<br/>Runtime"]
    SEG_RT --> SSC_PROC["P_LoadSubsectors<br/>Wire pointers"]
    SSC_PROC --> SSC_RT["subsector_t[]<br/>Runtime"]
    
    S_RT --> SSC_PROC
    
    SSC_RT --> NOD_PROC["P_LoadNodes<br/>Wire pointers"]
    NOD_PROC --> NOD_RT["node_t[] BSP Tree<br/>Runtime"]
    
    NOD_RT --> RENDER["R_RenderPlayerView<br/>Traversal"]
    SEG_RT --> RENDER
    S_RT --> RENDER
    V_RT --> RENDER
    
    RENDER --> OUTPUT["Frame Rendered"]
```

---

## Diagram 7: Coordinate Spaces

```
World Space (map units, fixed-point 32-bit)
    ↓
View Space (relative to viewpoint, angles computed)
    ↓
Angle Space (0 to 2^32 = 360°)
    ↓
Screen Space (pixel column 0 to screenwidth-1)

Example transformation:
  World: vertex at x=12800 (fixed_t)
  View: relative to viewx=8192, angle=0
    → dx = 12800 - 8192 = 4608
  Angle: angle_t = R_PointToAngle(12800, vy)
    → angle_t result
  Screen: x = viewangletox[angle_index]
    → screen column (e.g., 150)
```

---

## Diagram 8: Memory Layout

```
Global Arrays (from doomstat):
  vertexes[numvertexes]     ← vertex_t
  lines[numlines]           ← line_t
  sides[numsides]           ← side_t
  sectors[numsectors]       ← sector_t
  segs[numsegs]             ← seg_t
  subsectors[numsubsectors] ← subsector_t
  nodes[numnodes]           ← node_t
  blockmap[]                ← spatial hash
  rejectmatrix[]            ← visibility lookup

Stack (frame-based):
  solidsegs[MAXSEGS]        ← Clip ranges
  drawsegs[MAXDRAWSEGS]     ← Draw records
  visplanes[MAXVISPLANES]   ← Plane cache
  vissprites[]              ← Sprite records

Heap (zone allocator):
  Various textures, sprites, misc objects
```

---

## Notes on Diagrams

- **Diagram 1** shows the initialization order and dependencies for BSP loading.
- **Diagram 2** is the main rendering loop; follows BSP traversal front-to-back.
- **Diagram 3** illustrates the tree structure and how nodes/subsectors are related.
- **Diagram 4** shows segment classification logic (the most complex part of rendering).
- **Diagram 5** is a sequence diagram showing the call order and interactions during frame rendering.
- **Diagram 6** shows data dependencies between WAD lumps and runtime structures.
- **Diagram 7** illustrates coordinate space transformations (world → screen).
- **Diagram 8** shows memory layout and global structures.

All diagrams are intended to be high-level; line-by-line code details are intentionally omitted.
