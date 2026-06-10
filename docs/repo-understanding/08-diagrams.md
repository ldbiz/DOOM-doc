# BSP usage diagrams

## 1. BSP data loading flow

```mermaid
flowchart LR
    A[DOOMWADDIR / detected IWAD] --> C[D_AddFile list]
    B[-file / -wart PWADs] --> C
    C --> D[W_InitMultipleFiles]
    D --> E[Global lump directory and cache]
    E --> F[G_DoLoadLevel]
    F --> G[P_SetupLevel finds map marker]
    G --> H[Load vertices, sectors, sides, lines]
    G --> I[Load subsectors, nodes, segs]
    G --> J[Cache REJECT and BLOCKMAP]
    H --> K[P_GroupLines]
    I --> K
    K --> L[Linked level-lifetime BSP/map state]
```

## 2. BSP runtime traversal and rendering flow

```mermaid
flowchart TD
    A[R_RenderPlayerView] --> B[Set camera and clear frame buffers]
    B --> C[R_RenderBSPNode at last node]
    C --> D{Node or subsector?}
    D -->|Node| E[Choose camera side]
    E --> F[Render front child]
    F --> G{Back child bbox may be visible?}
    G -->|Yes| H[Render back child]
    G -->|No| I[Prune back subtree]
    H --> D
    F --> D
    D -->|Subsector| J[Select sector floor and ceiling visplanes]
    J --> K[Add sector sprites]
    K --> L[Submit subsector segs]
    L --> M[Clip and draw visible wall ranges]
    M --> N[Accumulate drawsegs and plane bounds]
    C --> O[After traversal: draw planes]
    O --> P[Draw masked walls and sprites]
```

## 3. Key BSP/map structure relationships

```mermaid
erDiagram
    NODE ||--o{ NODE : child_may_reference
    NODE ||--o{ SUBSECTOR : child_may_reference
    NODE {
        fixed partition_line
        fixed child_bboxes
        ushort children
    }
    SUBSECTOR ||--|{ SEG : consecutive_range
    SUBSECTOR }o--|| SECTOR : assigned_to
    SEG }o--|| VERTEX : starts_at
    SEG }o--|| VERTEX : ends_at
    SEG }o--|| LINEDEF : derived_from
    SEG }o--|| SIDEDEF : faces_through
    SEG }o--|| SECTOR : front_sector
    SEG }o--o| SECTOR : back_sector
    LINEDEF }o--|| VERTEX : endpoint
    LINEDEF }o--o{ SIDEDEF : has_sides
    SIDEDEF }o--|| SECTOR : faces
```

`NODE` child relationships are encoded indexes, while most seg/map relationships are resolved to pointers at load time.

## 4. Sequence: render a player view through the BSP

```mermaid
sequenceDiagram
    participant Loop as Display loop
    participant Main as r_main.c
    participant BSP as r_bsp.c
    participant Segs as r_segs.c
    participant Planes as r_plane.c
    participant Things as r_things.c

    Loop->>Main: R_RenderPlayerView(player)
    Main->>Main: Set camera; clear clips, drawsegs, planes, sprites
    Main->>BSP: R_RenderBSPNode(root = numnodes - 1)
    loop Internal nodes
        BSP->>BSP: Choose viewer side and visit front child
        BSP->>BSP: Check opposite child bounding box against view/solid clips
    end
    loop Reached subsectors
        BSP->>Planes: Find floor and ceiling visplanes
        BSP->>Things: Add sprites from subsector sector
        BSP->>BSP: Add each seg and clip its screen range
        BSP->>Segs: Store/render visible wall ranges
        Segs->>Planes: Mark visible plane ranges
    end
    Main->>Planes: R_DrawPlanes()
    Main->>Things: R_DrawMasked() for masked walls and sprites
```

## 5. Gameplay spatial-query split

```mermaid
flowchart TD
    A[Gameplay spatial question] --> B{Question type}
    B -->|Which region contains this point?| C[R_PointInSubsector]
    C --> D[BSP single-branch descent]
    D --> E[Subsector and sector]
    B -->|Can object A see object B?| F[REJECT sector-pair check]
    F -->|Not rejected| G[BSP trace crossing nodes and subsectors]
    F -->|Rejected| H[Blocked]
    B -->|Movement / hitscan / use path candidates| I[P_PathTraverse or block iterators]
    I --> J[BLOCKMAP grid traversal]
```
