# Important Files and Directories

## Entrypoints

| File | Role | Why It Matters |
|------|------|----------------|
| `linuxdoom-1.10/i_main.c` | Platform-specific entry point | Sets up argument pointers and calls `D_DoomMain()` |
| `linuxdoom-1.10/d_main.c` | High-level initialization and main loop | Orchestrates startup, calls all subsystem initializers, runs the game loop |
| `linuxdoom-1.10/Makefile` | Build configuration | Defines compilation flags, object files, and linking rules |

## Core Architecture & State

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/doomdef.h` | Global type definitions and constants | Defines fundamental types (`mobj_t`, `sector_t`, `player_t`, `GameMode_t`, etc.); defines DOOM version and game mode (shareware/registered/commercial) |
| `linuxdoom-1.10/doomstat.h` | Global game state variables | Centralizes all global state (players, game mode, gametic, demo flags) |
| `linuxdoom-1.10/d_main.h` | Main loop interface | Documents the main entry point and WAD file interface |
| `linuxdoom-1.10/doomdata.h` | WAD file format structures | Defines binary format for map data (vertices, sectors, linedefs, etc.) |

## Game Logic

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/g_game.c` | Game state machine and core game rules | Manages game progression (loading levels, demos, saves), player state, network sync checks, and the fundamental game loop ticker |
| `linuxdoom-1.10/d_net.c` | Network protocol and synchronization | Implements lock-step network protocol; manages network tick buffering and consistency checks |
| `linuxdoom-1.10/d_event.h` | Event types | Defines input event structure for keyboard/mouse/network input |

## Physics & Map

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/p_setup.c` | Map loading and initialization | Loads map data from WAD file, builds runtime structures (BSP tree, sector lists, thing spawning) |
| `linuxdoom-1.10/p_map.c` | Collision detection and movement | Implements line-of-sight testing, movement traces, and the core physics query functions |
| `linuxdoom-1.10/p_maputl.c` | Map utility functions | BSP traversal, blockmap queries, visibility checks |
| `linuxdoom-1.10/p_mobj.c` | Object (mobj) lifecycle | Creates, updates, and destroys movable objects; handles object thinking and state machines |
| `linuxdoom-1.10/p_enemy.c` | Monster AI | Implements monster behavior: movement, targeting, decision-making |
| `linuxdoom-1.10/p_inter.c` | Interaction & damage | Handles weapon hits, item pickups, monster attacks |
| `linuxdoom-1.10/p_sight.c` | Line-of-sight testing | Determines if monsters can "see" the player; used for targeting and AI |
| `linuxdoom-1.10/p_spec.c` | Special map sectors and effects | Door/floor/ceiling mechanics, switches, teleporters, damaging sectors |
| `linuxdoom-1.10/p_tick.c` | Per-tick state updates | Calls all object thinkers each game tick |

## Rendering

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/r_main.c` | Rendering pipeline | Main rendering entry point; manages projection and camera state; calls subsystems to render visible world |
| `linuxdoom-1.10/r_bsp.c` | BSP tree traversal | Traverses the BSP tree to determine visible surfaces |
| `linuxdoom-1.10/r_segs.c` | Wall rendering | Draws walls using horizontal/vertical span algorithm |
| `linuxdoom-1.10/r_things.c` | Sprite rendering | Draws movable objects (monsters, pickups, decorations) as billboards |
| `linuxdoom-1.10/r_plane.c` | Floor/ceiling rendering | Draws horizontal surfaces |
| `linuxdoom-1.10/r_draw.c` | Low-level drawing | Pixel-level drawing functions for spans and sprites |
| `linuxdoom-1.10/r_data.c` | Texture and sprite data | Loads and manages textures, flats, and sprites from WAD file |
| `linuxdoom-1.10/tables.c` | Pre-computed lookup tables | Sine/cosine tables, visibility masks, light falloff tables |

## UI & HUD

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/st_stuff.c` | Status bar | Renders HUD with health, ammo, armor; triggers screen wiping effects |
| `linuxdoom-1.10/hu_stuff.c` | Head-up display | Intermission screens, level end statistics, messages |
| `linuxdoom-1.10/am_map.c` | Automap | In-game map display with player/monster positions |
| `linuxdoom-1.10/m_menu.c` | Main menu & options | Menu system for game start, skill selection, options |

## Platform Abstraction

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/i_video.c` | Graphics output | X11-based rendering; manages framebuffer and screen updates |
| `linuxdoom-1.10/i_sound.c` | Sound output | Interface to sound device (stub in main engine; may delegate to `sndserv`) |
| `linuxdoom-1.10/i_net.c` | Networking | UDP socket creation and packet transmission/reception |
| `linuxdoom-1.10/i_system.c` | System utilities | Timing, error handling, system calls |

## Audio (Sound Server)

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `sndserv/soundsrv.c` | Sound server main loop | Standalone process for audio playback; reads sound commands from WAD file interface |
| `sndserv/sounds.c` | Sound effect management | Manages sound channels and playback state |
| `sndserv/wadread.c` | WAD reading for audio | Loads sound data from WAD files |
| `sndserv/Makefile` | Sound server build | Compilation rules for `sndserv` executable |

## Network Protocol Support

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `sersrc/` | Serial port networking | Source code for serial-link multiplayer (not linked into main game) |
| `ipx/` | IPX network support | Source code for IPX network layer (not linked into main game) |

## Utilities & Data

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/m_argv.c` | Argument parsing | Parses command-line options |
| `linuxdoom-1.10/m_misc.c` | Configuration files | Loads/saves DOOM.CFG configuration |
| `linuxdoom-1.10/m_random.c` | Random number generation | Deterministic RNG for network sync |
| `linuxdoom-1.10/m_fixed.c` | Fixed-point arithmetic | Fixed-point math utilities |
| `linuxdoom-1.10/m_bbox.c` | Bounding box utilities | Geometry calculations for collision/visibility |
| `linuxdoom-1.10/z_zone.c` | Memory management | Zone-based memory allocator |
| `linuxdoom-1.10/w_wad.c` | WAD file I/O | Reads game data from WAD files |
| `linuxdoom-1.10/info.c` | Game object definitions | Massive auto-generated file defining all object types, sprites, and state machines |
| `linuxdoom-1.10/sounds.h` | Sound effect enumeration | Lists all sound effect IDs |
| `linuxdoom-1.10/d_englsh.h` | English strings | Text strings for menus, messages (also `d_french.h` for French) |

## Demo & Save System

| File / Directory | Role | Why It Matters |
|------------------|------|----------------|
| `linuxdoom-1.10/g_game.c` (demo functions) | Demo recording/playback | Records and replays player input for demo playback and network synchronization |
| `linuxdoom-1.10/p_saveg.c` | Save game serialization | Serializes/deserializes game state to disk |

## Documentation

| File | Role |
|------|------|
| `README.TXT` | Original release notes with architecture notes from John Carmack |
| `linuxdoom-1.10/README.asm` | Assembly optimization notes |
| `linuxdoom-1.10/README.book` | Book references for rendering techniques |
| `linuxdoom-1.10/README.sound` | Sound server setup documentation |
| `linuxdoom-1.10/README.gl` | OpenGL rendering notes |
| `linuxdoom-1.10/ChangeLog` | Version history and bug fixes |
| `linuxdoom-1.10/TODO` | Known issues and suggestions |
| `LICENSE.TXT` | Full GPL v2.0 license text |
