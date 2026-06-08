# Diagrams

This section contains Mermaid diagrams illustrating the architecture and key flows of the DOOM engine.

---

## Diagram 1: Runtime Flow (High-Level)

Shows the complete flow from program start to main game loop.

```mermaid
graph TD
    Start["Program Start<br/>i_main.c::main()"] -->|Initialize args| Init["d_main.c::D_DoomMain()"]
    
    Init -->|Memory| Z["Z_Init()"]
    Z -->|Parse args| Args["M_FindResponseFile()"]
    Args -->|Detect game| Version["IdentifyVersion()"]
    Version -->|Load data| WAD["W_InitMultipleFiles()"]
    
    WAD -->|Open display| Video["I_InitGraphics()"]
    Video -->|Init audio| Sound["S_Init()"]
    Sound -->|Init rendering| Render["R_Init()"]
    
    Render -->|Init subsystems| Sub["I_InitInput()<br/>M_Init()"]
    Sub -->|Load first map| Game["G_InitNew()"]
    
    Game -->|Main loop| Loop["D_DoomLoop()"]
    Loop -->|Get time| Time["I_GetTime()"]
    Time -->|Handle events| Events["D_ProcessEvents()"]
    Events -->|Tick ready?| Tick{Has tick time<br/>passed?}
    
    Tick -->|Yes| Advance["G_Ticker()"]
    Tick -->|No| Draw1["Skip game tick"]
    
    Advance -->|Update state| Physics["P_Ticker()"]
    Physics -->|Render frame| Render1["R_RenderPlayerView()"]
    Draw1 -->|Render frame| Render1
    
    Render1 -->|Update HUD| HUD["ST_Drawer()"]
    HUD -->|Send to display| Screen["V_ScreenUpdate()"]
    Screen -->|Loop| Loop
```

---

## Diagram 2: Module Interaction (Core Subsystems)

Shows how major subsystems interact.

```mermaid
graph TB
    Start["Game Loop<br/>d_main.c::D_DoomLoop()"]
    
    Start --> GameState["Game State<br/>g_game.c<br/>doomstat.h"]
    Start --> Input["Input<br/>g_game.c::G_BuildTiccmd()"]
    Start --> Network["Network<br/>d_net.c"]
    
    Input --> Physics["Physics & World<br/>p_*.c"]
    Network --> Physics
    GameState --> Physics
    
    Physics --> Objects["Object Lifecycle<br/>p_mobj.c"]
    Physics --> Collision["Collision Detection<br/>p_map.c"]
    Physics --> AI["Monster AI<br/>p_enemy.c"]
    
    Physics --> Render["Rendering<br/>r_*.c"]
    GameState --> Render
    
    Render --> Map["Map Data<br/>p_setup.c"]
    Render --> BSP["BSP Traversal<br/>r_bsp.c"]
    Map --> BSP
    
    Render --> Screen["Video Output<br/>i_video.c<br/>v_video.c"]
    
    Physics --> Sound["Audio<br/>s_sound.c"]
    Sound --> SoundDevice["Sound Output<br/>i_sound.c<br/>sndserv/"]
    
    GameState --> UI["UI/HUD<br/>st_stuff.c<br/>hu_stuff.c<br/>m_menu.c"]
    UI --> Screen
    
    Map --> WAD["WAD System<br/>w_wad.c"]
    WAD --> Texture["Texture/Sprite Data<br/>r_data.c"]
    Texture --> Render
    
    GameState --> Demo["Demo System<br/>g_game.c"]
    Input --> Demo
    Physics --> Demo
```

---

## Diagram 3: Game Tick Sequence

Shows what happens during a single game tick (35 Hz, ~28 ms).

```mermaid
sequenceDiagram
    participant FrameTime as Frame Timing
    participant Input as Input Handling
    participant Net as Network Sync
    participant Physics as Physics Update
    participant Render as Rendering
    participant AI as AI/Thinkers
    
    FrameTime ->> Input: Check if tick time ready
    Input ->> Input: Poll keyboard/mouse
    Input ->> Physics: Build ticcmd from input
    
    alt Network Game
        Physics ->> Net: Send ticcmd
        Net ->> Net: Wait for all nodes
        Net ->> Physics: Provide remote ticcmds
    end
    
    Physics ->> AI: P_Ticker() - run all thinkers
    AI ->> AI: Monster AI (A_Chase, A_Look)
    AI ->> AI: Projectile movement
    AI ->> AI: Sector effects (doors, platforms)
    AI ->> Physics: Update positions, check collisions
    Physics ->> Physics: Apply gravity, velocity
    
    Physics ->> Render: Frame ready - set viewpoint
    Render ->> Render: BSP traversal (R_RenderPlayerView)
    Render ->> Render: Draw walls, floors, sprites
    Render ->> Render: Compose HUD (status bar, messages)
    Render ->> FrameTime: Display frame
    
    FrameTime ->> FrameTime: Wait for next tick
```

---

## Diagram 4: Physics and Collision Query

Shows how collision detection and physics queries work.

```mermaid
graph LR
    Mobj["Object<br/>Position, Velocity"]
    
    Mobj -->|Try to move| CheckPos["P_CheckPosition()"]
    CheckPos -->|Query blockmap| Blockmap["Blockmap Query<br/>Find cells containing obj"]
    Blockmap -->|Get lines in cells| LineQuery["Get linedefs<br/>intersecting path"]
    
    LineQuery -->|Check each line| InterceptTest["Test line intercept<br/>with move path"]
    InterceptTest -->|If blocked| Blocked["Blocked result"]
    InterceptTest -->|If clear| NextLine["Check next line"]
    
    Blockmap -->|Get mobjs in cells| ObjQuery["Get mobjs<br/>in nearby cells"]
    ObjQuery -->|Check collision| ObjTest["Check mobj distance<br/>radius overlaps"]
    ObjTest -->|If collision| ObjBlocked["Collision result"]
    ObjTest -->|If clear| NextObj["Check next mobj"]
    
    Blocked -->|Report| Result["Move blocked<br/>Position unchanged"]
    ObjBlocked -->|Report| Result
    NextLine -->|Done| Clear["Move allowed<br/>Position updated"]
    NextObj -->|Done| Clear
    
    Clear -->|For ranged attack| Sight["P_CheckSight()<br/>Line-of-sight test"]
    Sight -->|BSP traversal| SightBSP["Traverse BSP<br/>Check line-of-sight"]
    
    Mobj -->|Firing hitscan| Aim["P_AimLineAttack()"]
    Aim -->|Trace line| Intercepts["Gather intercepts<br/>with walls/mobjs"]
    Intercepts -->|Find first hit| Target["First target found"]
    Target -->|Apply damage| Damage["P_DamageMobj()"]
```

---

## Diagram 5: Monster AI State Machine

Shows how a monster transitions through behavioral states.

```mermaid
stateDiagram-v2
    [*] --> Idle
    
    Idle -->|See player| Chase
    Idle -->|Hear sound| Alert
    Alert -->|See player| Chase
    Alert -->|No player| Idle
    
    Chase -->|In melee range| Melee
    Chase -->|In ranged range| Ranged
    Chase -->|Lost player| Idle
    
    Melee -->|Attack ready| Punch
    Punch -->|Attacked| Chase
    Punch -->|Lost player| Idle
    
    Ranged -->|Projectile ready| Fire
    Fire -->|Fired| Chase
    Fire -->|Lost player| Idle
    
    Chase -->|Take damage| Pain
    Pain -->|Recover| Chase
    
    Chase -->|Health <= 0| Die
    Melee -->|Health <= 0| Die
    Ranged -->|Health <= 0| Die
    Idle -->|Health <= 0| Die
    
    Die --> [*]
    
    note right of Chase
        P_MapThings.A_Chase()
        Calculates direction to player
        Tries to move closer
        Plays animation frame
    end
    
    note right of Fire
        P_Enemy.A_BruiserAttack() etc.
        P_SpawnMissile() spawns projectile
        Projectile thinker runs each tick
    end
```

---

## Diagram 6: Rendering Pipeline

Shows the complete rendering flow from world space to screen pixels.

```mermaid
graph TD
    Start["R_RenderPlayerView()"]
    
    Start -->|Setup| Setup["Set viewpoint<br/>Calculate projection"]
    Setup -->|Clear screen| Clear["Clear framebuffer"]
    Clear -->|BSP| BSP["R_RenderBSPNode()"]
    
    BSP -->|Traverse| Traverse["Walk BSP tree<br/>front-to-back from camera"]
    Traverse -->|At leaf| Subsector["In subsector"]
    Subsector -->|Draw visible| DrawSeg["R_RenderSegRange()"]
    
    DrawSeg -->|For each seg| Seg["Wall segment"]
    Seg -->|Clip to view| Clip["Clip to screen boundaries"]
    Clip -->|Vertical span| Span["Draw vertical span<br/>from ceiling to floor"]
    Span -->|Lookup texture| Texture["Get wall texture from WAD"]
    Texture -->|Scale & light| Light["Apply lighting based<br/>on distance/sector"]
    Light -->|Draw pixels| Pixel["Write pixels to buffer<br/>R_DrawColumn()"]
    
    Subsector -->|Draw floors/ceilings| Planes["R_DrawPlanes()"]
    Planes -->|For each plane| Plane["Floor or ceiling"]
    Plane -->|Horizontal span| HSpan["Draw horizontal span<br/>R_DrawSpan()"]
    HSpan -->|Get flat| Flat["Get floor/ceiling flat"]
    Flat -->|Light& render| PLight["Apply lighting"]
    PLight -->|Draw pixels| PPixel["Write to buffer"]
    
    Subsector -->|Collect sprites| Sprites["Gather sprites in subsector"]
    Sprites -->|For each sprite| Sprite["Sprite (mobj)"]
    Sprite -->|Project| Project["Project to screen<br/>Calculate screen bounds"]
    Project -->|Clip| SClip["Clip to wall columns"]
    SClip -->|Get sprite frame| Frame["Get sprite frames<br/>from WAD"]
    Frame -->|Draw columns| SCol["R_DrawVisSprite()"]
    SCol -->|For each column| Draw["Draw sprite column<br/>Check occlusion"]
    Draw -->|Write pixels| SPixel["Write to buffer"]
    
    Traverse -->|Continue| TraverseCont["Next subsector"]
    TraverseCont -->|Done| Final["All subsectors rendered"]
    
    Final -->|HUD| HUD["ST_Drawer()<br/>Render status bar"]
    HUD -->|Messages| MSG["HU_Drawer()<br/>Draw messages"]
    MSG -->|Display| Display["V_ScreenUpdate()<br/>Send buffer to X11"]
```

---

## Diagram 7: Network Synchronization

Shows how networked games stay in sync.

```mermaid
graph LR
    Player1["Node A<br/>Player 1"]
    Player2["Node B<br/>Player 2"]
    Player3["Node C<br/>Player 3"]
    
    Player1 -->|Tick N| Ticcmd1A["Build ticcmd"]
    Player2 -->|Tick N| Ticcmd2A["Build ticcmd"]
    Player3 -->|Tick N| Ticcmd3A["Build ticcmd"]
    
    Ticcmd1A -->|Send to all| Net1["UDP Broadcast<br/>Ticcmd A"]
    Ticcmd2A -->|Send to all| Net2["UDP Broadcast<br/>Ticcmd B"]
    Ticcmd3A -->|Send to all| Net3["UDP Broadcast<br/>Ticcmd C"]
    
    Net1 -->|Recv| Buffer1["Node A: Buffer all ticcmds"]
    Net2 -->|Recv| Buffer1
    Net3 -->|Recv| Buffer1
    
    Net1 -->|Recv| Buffer2["Node B: Buffer all ticcmds"]
    Net2 -->|Recv| Buffer2
    Net3 -->|Recv| Buffer2
    
    Net1 -->|Recv| Buffer3["Node C: Buffer all ticcmds"]
    Net2 -->|Recv| Buffer3
    Net3 -->|Recv| Buffer3
    
    Buffer1 -->|Have all?| Sync1{"Check:<br/>Have ticcmd<br/>from all nodes<br/>for Tick N?"}
    Buffer2 -->|Have all?| Sync2{"Check:<br/>Have ticcmd<br/>from all nodes<br/>for Tick N?"}
    Buffer3 -->|Have all?| Sync3{"Check:<br/>Have ticcmd<br/>from all nodes<br/>for Tick N?"}
    
    Sync1 -->|Yes| Advance1["Advance tick<br/>Apply all ticcmds"]
    Sync2 -->|Yes| Advance2["Advance tick<br/>Apply all ticcmds"]
    Sync3 -->|Yes| Advance3["Advance tick<br/>Apply all ticcmds"]
    
    Sync1 -->|No| Wait1["Wait/Retry"]
    Sync2 -->|No| Wait2["Wait/Retry"]
    Sync3 -->|No| Wait3["Wait/Retry"]
    
    Advance1 -->|Determin.| Same1["Same game state<br/>across all nodes"]
    Advance2 -->|Determin.| Same2["Same game state<br/>across all nodes"]
    Advance3 -->|Determin.| Same3["Same game state<br/>across all nodes"]
    
    Same1 -.->|All identical| Result["✓ Network sync<br/>All nodes in sync"]
    Same2 -.->|All identical| Result
    Same3 -.->|All identical| Result
```

---

## Diagram 8: Memory Architecture

Shows how memory is organized and managed.

```mermaid
graph TD
    Heap["System Heap (malloc)"]
    
    Heap -->|Z_Init()| Zone["Zone Allocator<br/>z_zone.c"]
    
    Zone -->|Allocate| MainZone["Main Zone<br/>Fixed size buffer"]
    
    MainZone -->|W_InitMultipleFiles()| WADCache["WAD Cache<br/>Game data"]
    MainZone -->|R_Init()| ScreenBuf["Screen Buffer<br/>320x200 pixels"]
    MainZone -->|R_Init()| Textures["Texture Cache<br/>All textures"]
    MainZone -->|R_Init()| Sprites["Sprite Cache<br/>All sprites"]
    MainZone -->|P_LoadThings()| MapData["Map Data<br/>Sectors, linedefs"]
    MainZone -->|G_InitNew()| Mobjs["Active Mobjs<br/>Players, monsters"]
    
    WADCache -->|Lookup| Lumps["Lump Directory"]
    Lumps -->|Read| Lump1["Lump 1<br/>Map data"]
    Lumps -->|Read| Lump2["Lump 2<br/>Sprite data"]
    Lumps -->|Read| LumpN["Lump N<br/>..."]
    
    MapData -->|Contains| Vertexes["Vertices"]
    MapData -->|Contains| Linedefs["Linedefs"]
    MapData -->|Contains| Sectors["Sectors"]
    MapData -->|Contains| Things["Things<br/>spawn points"]
    
    Mobjs -->|Contains| Player["Player 1-4"]
    Mobjs -->|Contains| Monsters["Monsters"]
    Mobjs -->|Contains| Projectiles["Projectiles"]
    Mobjs -->|Contains| Pickups["Pickups"]
    
    ScreenBuf -->|320x200x1| Pixels["8-bit pixels<br/>palette indices"]
```

---

## Diagram 9: WAD File System

Shows how the WAD file system is organized and accessed.

```mermaid
graph LR
    Game["Game Start<br/>main()"]
    Game -->|Find& load| IWAD["IWAD<br/>DOOM.WAD or DOOM2.WAD<br/>~2-4 MB"]
    Game -->|Optional| PWAD1["PWAD 1<br/>Mod/addon<br/>~100 KB"]
    Game -->|Optional| PWAD2["PWAD 2<br/>Mod/addon<br/>~200 KB"]
    
    IWAD -->|W_AddFile()| Directory["Master Lump Directory<br/>All WAD files merged"]
    PWAD1 -->|W_AddFile()| Directory
    PWAD2 -->|W_AddFile()| Directory
    
    Directory -->|Hash table| Lookup["Fast O(1) Lump Lookup<br/>W_CheckNumForName()"]
    
    Lookup -->|By name| Map["MAP01 lump<br/>Map data"]
    Lookup -->|By name| Sprite["TROOA0 lump<br/>Sprite frame"]
    Lookup -->|By name| Texture["STARTAN1 lump<br/>Wall texture"]
    Lookup -->|By name| Flat["STONE lump<br/>Floor/ceiling"]
    Lookup -->|By name| Sound["DSPISTOL lump<br/>Sound data"]
    Lookup -->|By name| Palette["PLAYPAL lump<br/>Color palette"]
    
    Map -->|P_LoadLevel()| MapData["Parse map data<br/>Create world geometry"]
    Sprite -->|R_InitSprites()| SpriteCache["Load into memory<br/>Cache sprites"]
    Texture -->|R_InitTextures()| TexCache["Load into memory<br/>Cache textures"]
    Flat -->|R_InitFlats()| FlatCache["Load into memory<br/>Cache flats"]
    Sound -->|S_Init()| SoundCache["Load sound data<br/>or pass to sndserv"]
    Palette -->|I_InitGraphics()| PalInit["Initialize palette<br/>X11 colormap"]
```

---

## Diagram 10: Player Input to Action

Shows how keyboard/mouse input becomes in-game action.

```mermaid
graph LR
    Keyboard["Keyboard Event<br/>e.g., 'W' key pressed"]
    Mouse["Mouse Event<br/>e.g., button click"]
    
    Keyboard -->|X11 event| EventQueue["Event Queue"]
    Mouse -->|X11 event| EventQueue
    
    EventQueue -->|D_ProcessEvents()| Dispatch["Dispatch events to handlers"]
    
    Dispatch -->|Key event| KeyBind["Lookup key binding<br/>from config"]
    Dispatch -->|Mouse event| MouseBind["Lookup mouse binding"]
    
    KeyBind -->|Map to action| Forward["Forward movement"]
    KeyBind -->|Map to action| Turn["Turn"]
    KeyBind -->|Map to action| Fire["Fire weapon"]
    KeyBind -->|Map to action| Use["Use item"]
    
    MouseBind -->|Map to action| Look["Look/aim"]
    MouseBind -->|Map to action| MFire["Fire weapon"]
    
    Forward -->|G_BuildTiccmd()| Ticcmd["Build ticcmd<br/>Tick command"]
    Turn -->|G_BuildTiccmd()| Ticcmd
    Fire -->|G_BuildTiccmd()| Ticcmd
    Use -->|G_BuildTiccmd()| Ticcmd
    Look -->|G_BuildTiccmd()| Ticcmd
    MFire -->|G_BuildTiccmd()| Ticcmd
    
    Ticcmd -->|Network/Local| Execute["Execute ticcmd"]
    Execute -->|P_Ticker()| Action["Apply action in game"]
    Action -->|Forward→| Move["Move player forward"]
    Action -->|Turn→| Rotate["Rotate player view"]
    Action -->|Fire→| Weapon["Fire current weapon"]
    Action -->|Use→| Interact["Interact with world"]
```

These diagrams provide visual understanding of the system's architecture, data flow, and key algorithms. They complement the textual documentation in the other files.
