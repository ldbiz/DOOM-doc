# Core Concepts

## Game Concepts

### Mobj (Moving Object)

**Meaning**: A mobj (defined in `p_mobj.h`) is any movable or interactive object in the game world: players, monsters, projectiles, pickups, decorations.

**Key Properties**:
- Position, velocity, angle, height
- Type (determined from `info.c` definitions)
- State machine for animation and behavior
- Health, damage, special flags

**Where It Appears**:
- Core structure in `linuxdoom-1.10/p_mobj.h`
- Created in `p_setup.c::P_SpawnMapThing()` during map load
- Updated each tick in `p_mobj.c::P_MobjThinker()`
- Physics and collision in `p_map.c`, `p_maputl.c`

**Related Files**: `p_mobj.c`, `p_inter.c`, `p_enemy.c`

---

### Sector

**Meaning**: A sector is a convex area of the map with uniform floor and ceiling heights, lighting, and special effects (damage, teleport, etc.). Every point in the map belongs to exactly one sector.

**Key Properties**:
- Floor height, ceiling height
- Lightlevel
- Floor and ceiling flat (texture)
- Special sector type (damaging lava, exit floor, etc.)

**Where It Appears**:
- Defined in `linuxdoom-1.10/doomdata.h` (binary WAD format)
- Parsed into runtime struct in `p_setup.c`
- Physics queries in `p_map.c`, `p_sight.c`
- Rendering in `r_plane.c`, `r_segs.c`

**Related Files**: `p_setup.c`, `p_map.c`, `r_plane.c`

---

### Linedef (Line Definition)

**Meaning**: A linedef is a map wall segment with two endpoints (vertices) and associated textures. It defines the impassable boundaries of the map geometry.

**Key Properties**:
- Start and end vertices
- Front and back sidedefs (textures on each side)
- Flags (impassable, two-sided, etc.)
- Special type (door, elevator, teleporter, etc.)
- Activation tags and arguments

**Where It Appears**:
- Defined in `linuxdoom-1.10/doomdata.h`
- Parsed in `p_setup.c`
- Collision queries in `p_map.c`, `p_maputl.c`
- Rendering in `r_segs.c`

**Related Files**: `p_setup.c`, `p_map.c`, `r_segs.c`

---

### BSP Tree (Binary Space Partition)

**Meaning**: A hierarchical spatial data structure that recursively divides the map to determine which surfaces are visible from a given viewpoint. Essential for efficient rendering.

**How It Works**:
- Pre-built during map load; stored in WAD file
- Each node represents a line that divides space into front/back
- Leaves (subsectors) contain lists of visible walls and sprites
- Rendering traverses the tree front-to-back from the camera

**Where It Appears**:
- Loaded in `p_setup.c::P_LoadNodes()`
- Rendered in `r_bsp.c::R_RenderBSPNode()`
- Collision queries in `p_maputl.c`

**Related Files**: `p_setup.c`, `r_bsp.c`, `p_maputl.c`

---

### Tic (Game Tick)

**Meaning**: A discrete time step in the game simulation. DOOM runs at 35 Hz (one tic every ~28 ms). All game state updates happen once per tic.

**Properties**:
- Fixed duration (~28 milliseconds)
- All players and AI advance exactly one tic per network synchronization frame
- Deterministic: same inputs produce same outputs (required for network multiplayer)

**Where It Appears**:
- Central variable: `gametic` in `g_game.c`
- Main loop in `d_main.c::D_DoomLoop()`
- Ticker functions called in `p_tick.c::P_Ticker()`

**Related Files**: `d_main.c`, `g_game.c`, `p_tick.c`

---

### Sprite

**Meaning**: A pre-drawn 2D image representing an object (monster, pickup, decoration, projectile). Sprites are drawn as billboards (always facing the camera).

**Properties**:
- Multiple frames for animation
- Rotations (8 viewing angles, or single frame)
- Stored as columns of pixels (vertical spans for fast drawing)

**Where It Appears**:
- Defined in WAD file
- Loaded in `r_data.c::R_InitSprites()`
- Drawn in `r_things.c`

**Related Files**: `r_data.c`, `r_things.c`

---

### Flat

**Meaning**: A flat is a repeating texture used for floors and ceilings. Unlike walls, flats tile across the entire surface.

**Where It Appears**:
- Stored in WAD file
- Loaded in `r_data.c`
- Rendered in `r_plane.c`

**Related Files**: `r_data.c`, `r_plane.c`

---

### Thinker

**Meaning**: A thinker is a function pointer associated with an mobj that gets called each game tick to update the object's state. Implements state machines for animation and behavior.

**Usage**:
- All active mobjs have a thinker function
- Called in `p_tick.c::P_Ticker()`
- Examples: `A_Chase` (monster AI), `A_Look` (idle), projectile movement functions

**Where It Appears**:
- Thinker list maintained globally
- Set in `info.c` state table
- Called in `p_tick.c`

**Related Files**: `p_tick.c`, `p_mobj.c`, `p_enemy.c`, `info.c`

---

### State Machine (from info.c)

**Meaning**: Each mobj type has a sequence of animation frames and associated actions. The state machine defines transitions, frames per state, and callback functions.

**Structure**:
- Each state has: sprite, frame, duration, action function, next state
- Defined in large table in `info.c`
- Provides behavior scripting without code modification

**Where It Appears**:
- Massive auto-generated `info.c` file
- State transitions in `p_mobj.c::P_SetMobjState()`
- State-specific actions called during tick

**Related Files**: `info.c`, `info.h`, `p_mobj.c`

---

### Subsector

**Meaning**: A subsector is a convex region of the map (a leaf in the BSP tree) bounded by map lines. All points in a subsector can see the same set of surfaces.

**Purpose**:
- Defines the scope of visibility
- Associates sprite/thing lists with regions
- Rendering renders one subsector's surfaces at a time

**Where It Appears**:
- Created during map load in `p_setup.c`
- Traversed in rendering in `r_bsp.c`

**Related Files**: `p_setup.c`, `r_bsp.c`

---

### Intercept

**Meaning**: An intercept is a point where a ray (or line segment) intersects a linedef. Used for collision detection and line-of-sight testing.

**Where It Appears**:
- Ray-casting in `p_maputl.c` and `p_sight.c`
- Used in `p_map.c` for collision detection

**Related Files**: `p_map.c`, `p_maputl.c`, `p_sight.c`

---

### Gametic vs Maketic

**Meaning**:
- `gametic`: The current game tick being rendered/displayed
- `maketic`: The next tick to be simulated (used in network synchronization to track which ticks have player input)

**Purpose**: In networked games, a player may prepare input for a tick before all remote players' input has arrived. This distinction allows the engine to buffer input and synchronize tick by tick.

**Where It Appears**:
- Both in `g_game.c`
- Used in network tick buffer in `d_net.c`

**Related Files**: `d_net.c`, `g_game.c`

---

### Demo

**Meaning**: A recorded sequence of player inputs (ticcmds) that can be replayed to recreate a game session. Used for:
- Attract mode demonstrations
- Competitive replays
- Network synchronization debugging

**Where It Appears**:
- Recording in `g_game.c::G_WriteDemoTiccmd()`
- Playback in `g_game.c::G_ReadDemoTiccmd()`
- Buffer in `g_game.c::demobuffer`

**Related Files**: `g_game.c`, `p_saveg.c`

---

### WAD (Where's All the Data)

**Meaning**: WAD is the DOOM file format containing all game assets: maps, sprites, textures, sounds, music, flat patterns. An IWAD is an "internal WAD" (the main game data); a PWAD is a "patch WAD" (mod/addon).

**Structure**:
- Header with directory
- Lumps (named data blocks)
- Directory of lump locations
- Defined in `doomdata.h`

**Where It Appears**:
- Loaded in `w_wad.c`
- Referenced throughout: textures in `r_data.c`, sprites in `r_data.c`, maps in `p_setup.c`, sounds in `s_sound.c`

**Related Files**: `w_wad.c`, `doomdata.h`

---

### Ticcmd (Tick Command)

**Meaning**: A ticcmd is the player input for a single game tick: forward/back movement, strafing, turning, button presses (shoot, open door, use).

**Purpose**: In networked games, each player sends their ticcmd to all other players each tick, allowing remote players to simulate that player's movement and actions deterministically.

**Where It Appears**:
- Structure in `d_ticcmd.h`
- Built in `g_game.c::G_BuildTiccmd()`
- Sent in `d_net.c`

**Related Files**: `d_ticcmd.h`, `d_net.c`, `g_game.c`

---

### Fixed-Point Arithmetic

**Meaning**: A number representation where the binary point is fixed at a specific position (typically 16 bits to the right). Used for fast math in integer CPUs before floating-point hardware was standard.

**Type**: `fixed_t` in `m_fixed.h`

**Where It Appears**:
- All object positions and velocities use `fixed_t`
- Math utilities in `m_fixed.c`
- Critical for deterministic network synchronization

**Related Files**: `m_fixed.h`, `m_fixed.c`

---

### LUT (Look-Up Table)

**Meaning**: A pre-computed table of trigonometric or lighting values to avoid expensive per-frame calculations.

**Examples**:
- `finesine`, `finecosine` in `tables.c` — pre-computed sine/cosine at high precision
- `scalelight` — pre-computed light falloff curves
- `colormaps` — lighting variations for different light levels

**Where It Appears**:
- Generated/loaded in `tables.c` and `r_data.c`
- Used throughout rendering pipeline

**Related Files**: `tables.c`, `r_data.c`

---

### Blockmap

**Meaning**: A grid overlay on the map used for efficient spatial queries. Speeds up collision detection and line-of-sight tests by avoiding checking all lines/things in the world.

**How It Works**:
- Map is divided into square cells
- Lines and things are indexed by which cells they overlap
- Query: find cell containing a point, check only lines/things in that cell

**Where It Appears**:
- Created in `p_setup.c::P_CreateBlockMap()`
- Queried in `p_map.c` and `p_maputl.c`

**Related Files**: `p_setup.c`, `p_map.c`, `p_maputl.c`

---

### Hitscan

**Meaning**: A hit-scan attack is an instant-hit attack (like a gun) that traces a line from the attacker to check for intersections with targets. Opposite of projectile weapons that follow a trajectory.

**Where It Appears**:
- Weapon firing in `p_pspr.c`
- Line tracing in `p_map.c::P_AimLineAttack()`

**Related Files**: `p_pspr.c`, `p_map.c`

---

### PVS (Potentially Visible Set)

**Meaning**: Although not explicitly calculated or stored in this DOOM version, visibility is determined implicitly through BSP traversal and intercept testing. Surfaces in the same subsector are always visible; cross-subsector visibility is determined by line-of-sight.

**Where It Appears**:
- BSP traversal determines visible subsectors
- Line-of-sight testing in `p_sight.c`

**Related Files**: `r_bsp.c`, `p_sight.c`
