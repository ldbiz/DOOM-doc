# Change Map

This section provides a practical guide for common future changes. It identifies which files to look at first for each type of modification.

---

## If I Need to Change Startup/Runtime Behavior

**Start with**: `linuxdoom-1.10/d_main.c`

This file controls:
- Initialization order (`D_DoomMain()`)
- Main game loop (`D_DoomLoop()`)
- Subsystem initialization sequence

**Then look at**:
- `linuxdoom-1.10/i_main.c` — platform-specific entry point
- `linuxdoom-1.10/doomstat.h` — global game state
- `linuxdoom-1.10/g_game.c` — game state machine and tick logic

**Relevant subsystem files** (depending on what you're changing):
- `linuxdoom-1.10/i_video.c` — graphics initialization
- `linuxdoom-1.10/i_sound.c` — sound initialization
- `linuxdoom-1.10/i_net.c` — network initialization
- `linuxdoom-1.10/m_misc.c` — configuration loading

**Example changes**:
- Add new initialization step: insert in `D_DoomMain()` after existing inits
- Change tick rate: modify tick timing logic in `D_DoomLoop()`
- Add new subsystem: create `i_newsystem.c` and initialize in `D_DoomMain()`

---

## If I Need to Change Configuration

**Start with**: `linuxdoom-1.10/m_misc.c`

This file controls:
- Configuration file I/O (`M_LoadDefaults()`, `M_SaveDefaults()`)
- Default value initialization
- Per-setting validation

**Then look at**:
- `linuxdoom-1.10/doomdef.h` — define new setting constants
- `linuxdoom-1.10/d_main.c` — parse command-line arguments
- `linuxdoom-1.10/m_argv.c` — argument parsing utilities

**Where settings are used**:
- Key bindings: `linuxdoom-1.10/g_game.c`
- Graphics: `linuxdoom-1.10/i_video.c`
- Audio: `linuxdoom-1.10/s_sound.c`
- Game rules: `linuxdoom-1.10/g_game.c`, `linuxdoom-1.10/p_setup.c`

**Caveats**:
- Configuration format is binary (not human-editable); changing it breaks compatibility
- Command-line parsing happens before subsystem initialization, so early-stage decisions can depend on it
- Some settings are baked into compiled code (e.g., `SCREENWIDTH` in `doomdef.h`)

**Example changes**:
- Add new key binding: add field to config struct, load/save in `m_misc.c`, use in `g_game.c`
- Add new graphics setting: define in `doomdef.h`, add to config in `m_misc.c`, apply in `i_video.c`
- Add new difficulty modifier: parse in `d_main.c`, store in `doomstat.h`, use in `p_setup.c`/`p_enemy.c`

---

## If I Need to Change Request Handling or Core Behavior

**Depends on the behavior**:

### Player Input / Ticcmd Processing

**Start with**: `linuxdoom-1.10/g_game.c::G_BuildTiccmd()` and `linuxdoom-1.10/g_game.c::G_Ticker()`

- `G_BuildTiccmd()` converts keyboard/mouse input to a ticcmd
- `G_Ticker()` applies ticcmd to player state

**Then look at**:
- `linuxdoom-1.10/d_ticcmd.h` — ticcmd structure definition
- `linuxdoom-1.10/p_user.c` — player movement and action application
- `linuxdoom-1.10/p_pspr.c` — weapon firing logic

**Caveats**:
- Ticcmd changes affect network sync (all nodes must handle identically)
- Demo recording/playback must be compatible with new ticcmd format

**Example changes**:
- Add new action key: add flag to `ticcmd_t`, capture input in `G_BuildTiccmd()`, apply in `P_Ticker()`
- Change movement sensitivity: modify multiplier in `P_PlayerThink()`
- Add new weapon: define in `p_mobj.h`, spawn in `p_pspr.c`, add firing logic

### Map Loading / Geometry

**Start with**: `linuxdoom-1.10/p_setup.c`

This file controls:
- Map data parsing from WAD (`P_LoadVertexes()`, `P_LoadLinedefs()`, etc.)
- Spatial index creation (`P_CreateBlockMap()`, BSP tree loading)
- Object spawning (`P_SpawnMapThing()`)

**Then look at**:
- `linuxdoom-1.10/doomdata.h` — WAD file format structures
- `linuxdoom-1.10/w_wad.c` — WAD file I/O
- `linuxdoom-1.10/p_mobj.c` — object lifecycle
- `linuxdoom-1.10/p_spec.c` — sector special effects

**Caveats**:
- Map data is stored in WAD files (not in source code); changing parsing may require new WAD format
- BSP tree is pre-built by map editor; runtime code cannot rebuild it

**Example changes**:
- Add new sector special type: define constant in `doomdata.h`, handle in `p_spec.c::P_SpawnSpecials()`
- Add new object type: define in `info.h`/`info.c`, spawn in `P_SpawnMapThing()`
- Change collision detection: modify `P_CheckPosition()` in `p_map.c`

### Physics / Collision

**Start with**: `linuxdoom-1.10/p_map.c`

This file controls:
- Movement validation (`P_CheckPosition()`, `P_TryMove()`)
- Collision detection with walls and objects
- Hitscan/line-tracing (`P_AimLineAttack()`, `P_LineAttack()`)

**Then look at**:
- `linuxdoom-1.10/p_maputl.c` — map utility functions (blockmap queries, intercepts)
- `linuxdoom-1.10/p_sight.c` — line-of-sight testing
- `linuxdoom-1.10/m_bbox.c` — bounding box calculations

**Caveats**:
- Collision logic is tightly coupled to world geometry
- Fixed-point math must be preserved for network determinism
- Blockmap queries are heavily used; changes affect performance

**Example changes**:
- Change collision radius: modify checks in `P_CheckPosition()`
- Add new collision type: extend logic in `P_LineAttack()`
- Improve pathfinding: enhance `P_CheckPosition()` logic or blockmap usage

### Monster AI

**Start with**: `linuxdoom-1.10/p_enemy.c`

This file controls:
- Monster behavior logic (`A_Chase()`, `A_Look()`, etc.)
- Targeting and decision-making
- State transitions and attack timing

**Then look at**:
- `linuxdoom-1.10/info.c` — monster state definitions and animation sequences
- `linuxdoom-1.10/p_mobj.c` — object thinking/thinker management
- `linuxdoom-1.10/p_sight.c` — vision tests for targeting

**Caveats**:
- Monster behavior is deterministic (affects network sync)
- State machine in `info.c` is auto-generated; changing it requires regeneration
- Many monsters depend on action functions; changing one may affect multiple monster types

**Example changes**:
- Modify monster speed: change velocity in `A_Chase()`
- Change attack frequency: modify state duration in `info.c` state table
- Add new attack type: create new action function in `p_enemy.c`, call in state machine

### Rendering

**Start with**: `linuxdoom-1.10/r_main.c`

This file controls:
- Rendering pipeline setup
- BSP traversal orchestration
- Camera/projection setup

**Then look at** (depending on what to change):
- `linuxdoom-1.10/r_bsp.c` — BSP tree traversal, visibility determination
- `linuxdoom-1.10/r_segs.c` — wall rendering
- `linuxdoom-1.10/r_plane.c` — floor/ceiling rendering
- `linuxdoom-1.10/r_things.c` — sprite rendering
- `linuxdoom-1.10/r_draw.c` — low-level pixel drawing
- `linuxdoom-1.10/r_data.c` — texture/sprite data management

**Caveats**:
- Rendering is performance-critical
- Fixed-point perspective math affects visual accuracy
- Palette-based 8-bit rendering limits color options
- Changes to projection may require lookup table recalculation

**Example changes**:
- Add transparency: extend `r_draw.c` drawing functions
- Change lighting: modify `r_data.c` colormap selection or `r_segs.c` wall shading
- Add new rendering feature: create function in appropriate `r_*.c` file and call from `r_main.c`

### HUD / UI

**Start with**: `linuxdoom-1.10/st_stuff.c` (status bar) or `linuxdoom-1.10/hu_stuff.c` (head-up display)

- Status bar: health, ammo, armor display
- Head-up display: messages, intermission screens
- Automap: `linuxdoom-1.10/am_map.c`

**Caveats**:
- HUD draws over game screen; must not affect gameplay
- Menu system has separate state machine (`linuxdoom-1.10/m_menu.c`)

**Example changes**:
- Change HUD layout: modify drawing code in `st_stuff.c`
- Add new status indicator: add field to player state, draw in `st_stuff.c`
- Change intermission screen: modify `wi_stuff.c`

---

## If I Need to Change External Service Integration

### Sound Output

**Start with**: `linuxdoom-1.10/s_sound.c` and `linuxdoom-1.10/i_sound.c`

- `s_sound.c` — game-side sound management (channels, effects)
- `i_sound.c` — platform interface (may spawn sndserv)

**Then look at**:
- `sndserv/soundsrv.c` — sound server implementation (if using separate process)
- `sndserv/sounds.c` — sound channel management in server
- `sndserv/wadread.c` — WAD file reading for sound data

**Caveats**:
- Sound can be stubbed out (no output)
- Sound server runs in separate process (complex inter-process communication)
- WAD file contains sound data; changing loading requires WAD format knowledge

**Example changes**:
- Disable sound: modify `I_InitSound()` to return without spawning sndserv
- Add new sound: define ID in `sounds.h`, trigger with `S_StartSound()`
- Change sound output device: modify `sndserv/linux.c` device initialization

### Network Communication

**Start with**: `linuxdoom-1.10/d_net.c` (protocol) and `linuxdoom-1.10/i_net.c` (I/O)

- `d_net.c` — game logic for synchronization and packet formatting
- `i_net.c` — UDP socket operations

**Then look at**:
- `ipx/` — IPX network layer (alternative to UDP)
- `sersrc/` — serial port layer (alternative to UDP)

**Caveats**:
- Network protocol must be deterministic (affects game state sync)
- Protocol changes break compatibility with old demos/saves
- Packet format is tightly coupled to game state

**Example changes**:
- Add new network feature: define in `doomdata_t` packet structure, handle in `d_net.c`
- Change transport layer: implement in new file (e.g., `i_newtransport.c`), call from `i_net.c`
- Add encryption/authentication: modify `NetSend()`/`NetListen()` in `i_net.c`

### WAD File Format

**Start with**: `linuxdoom-1.10/w_wad.c` and `linuxdoom-1.10/doomdata.h`

- `w_wad.c` — WAD file reading and lump lookup
- `doomdata.h` — WAD binary format structures

**Caveats**:
- WAD format is tightly coupled to external tools (map editors, texture browsers)
- Changing format breaks compatibility with existing WAD files
- Many file parsing routines depend on specific binary layout

**Example changes**:
- Add new lump type: define struct in `doomdata.h`, add loading code in `w_wad.c`
- Change lump lookup: modify hash/directory structure in `w_wad.c` (careful with performance)

---

## If I Need to Change UI/Templates/Static Assets

### Menus

**Start with**: `linuxdoom-1.10/m_menu.c`

This file controls:
- Menu structure and options
- Text rendering and positioning
- Input handling in menus

**Related files**:
- `linuxdoom-1.10/d_englsh.h` or `d_french.h` — menu strings (translations)
- `linuxdoom-1.10/st_lib.c` — HUD text drawing utilities

**Example changes**:
- Add new menu option: define in menu structure, add handler function
- Change menu layout: modify screen position calculations
- Add/change menu text: update language files

### In-Game Graphics (Sprites, Textures)

**Stored in**: WAD files (not in source code)

To change graphics:
- Edit WAD files with map editor or texture tool
- Graphics are loaded at runtime from WAD lumps
- Code references graphics by name (e.g., "TROOA0")

**Related source files**:
- `linuxdoom-1.10/r_data.c` — loads textures/sprites from WAD
- `linuxdoom-1.10/r_things.c` — renders sprites
- `linuxdoom-1.10/r_segs.c` — renders walls with textures

**Example changes**:
- Change how graphics are loaded: modify `R_InitTextures()`, `R_InitSprites()` in `r_data.c`
- Change graphics rendering: modify `r_things.c` or `r_segs.c`

---

## If I Need to Change Tests/Fixtures

**Note**: This repository has no automated test suite.

To add tests:
1. Create new test files or test runner
2. Use demo recording/playback as a testing mechanism
3. Record demo files as regression test fixtures
4. Automate playback to verify deterministic replay

**Related files**:
- `linuxdoom-1.10/g_game.c` — demo recording/playback logic
- `linuxdoom-1.10/p_saveg.c` — save game format (similar to demo format)

**Example changes**:
- Record new test demo: run game with `-record testname`
- Add playback test: run game with `-playback testname -timedemo` and check for crashes
- Add regression test suite: create shell script calling game with various `-playback` options

---

## Cross-Cutting Concerns

### Memory Management

Changes to memory allocation/deallocation:
- **File**: `linuxdoom-1.10/z_zone.c` (zone allocator)
- **Usage**: All dynamic allocation goes through `Z_Malloc()`, `Z_Free()`
- **Caveats**: Zone allocator is fast but not safe; changes affect all subsystems

### Fixed-Point Math

Changes to coordinate or velocity representation:
- **File**: `linuxdoom-1.10/m_fixed.c`
- **Usage**: All physics and rendering uses `fixed_t`
- **Caveats**: Changes affect determinism; network sync depends on exact same math results

### Look-Up Tables

Changes to pre-computed values:
- **File**: `linuxdoom-1.10/tables.c`
- **Usage**: Trigonometry, lighting curves, visibility masks
- **Caveats**: Changes affect both rendering quality and performance

### Global State

Changes to global variables:
- **File**: `linuxdoom-1.10/doomstat.h`, `doomstat.c`
- **Usage**: Game state, player state, netgame variables
- **Caveats**: Adding/removing globals affects network serialization and save game format
