# Repository Summary

## Purpose

This repository contains the original Linux source code release of DOOM v1.10, released by id Software on December 23, 1997. It is an intact archive of the game's engine used in the original DOOM and DOOM II, published under the GNU General Public License 2.0 for non-profit use and experimentation.

The release is a historical programming artifact documenting how one of the most influential first-person shooters was implemented. It includes the core game engine, rendering system, networking stack, and sound server infrastructure.

## Technology Stack

- **Language**: C (C++ style with static analysis compatible syntax)
- **Platform**: Linux (originally for Linux with 1990s-era dependencies)
- **Graphics**: X11 with custom 8-bit software rendering
- **Audio**: Dedicated sound server process communicating via WAD files
- **Networking**: UDP/IP sockets with IPX network layer support
- **Build System**: GNU Make
- **Dependencies**: X11 libraries (libX11, libXext), standard POSIX APIs

## Main Runtime Model

DOOM runs as a single-threaded event-driven game loop that coordinates multiple subsystems:

1. **Startup Phase**: The main entry point initializes the WAD file system (game data loading), X11 graphics, and networking infrastructure.
2. **Game Loop**: A central tick-based loop processes input, updates game state, and renders frames. The loop runs at a fixed 35 Hz tick rate.
3. **Separation of Concerns**: Input handling, game logic, rendering, and sound are logically separated, with a shared data model.
4. **Optional Sound Server**: Sound can be handled by a separate sound server process (`sndserv`), communicating through inter-process mechanisms.

The game supports both single-player and networked multi-player modes using a lock-step network protocol where all players synchronize their game state at each tick.

## Main External Services and Dependencies

- **X11 Display System**: Graphics output on Unix-like systems
- **WAD File Format**: Custom binary format containing game assets (maps, sprites, textures, sounds)
- **Optional IPX/Serial Networking**: For multi-player games (IPX network or serial port protocols)
- **Sound Hardware**: Audio via configured sound device (handled by `sndserv`)

## Main Entry Points

| Entry Point | Role |
|-----------|------|
| `linuxdoom-1.10/i_main.c::main()` | Platform entry point; calls `D_DoomMain()` |
| `linuxdoom-1.10/d_main.c::D_DoomMain()` | High-level initialization and starts the main game loop |
| `linuxdoom-1.10/d_main.c::D_DoomLoop()` | The main tick-based event loop (never exits during gameplay) |
| `linuxdoom-1.10/g_game.c::G_Ticker()` | Updates game state each tick |
| `sndserv/soundsrv.c` | Standalone sound server for audio output |

## Architecture Summary

The DOOM engine is organized into distinct layers with minimal dependencies between them:

**Foundation Layer** (`z_zone.c`, `w_wad.c`, `m_*.c`)
- Memory management with zone-based allocation
- WAD file loading (game asset data)
- Utility functions (math, random, argument parsing, fixed-point arithmetic)

**Platform Abstraction Layer** (`i_*.c`)
- Graphics (X11 rendering via `i_video.c`)
- Input handling (keyboard/mouse)
- Network communication (`i_net.c`)
- Sound interface (stub that may delegate to `sndserv`)

**Game State & Physics** (`p_*.c`)
- Map geometry and world state
- Object movement and collision detection
- Sprite interaction and damage
- Line-of-sight calculations
- Dynamic sector effects (doors, floors, ceilings)

**Rendering** (`r_*.c`)
- BSP tree traversal for visibility determination
- Wall/floor/ceiling rendering using horizontal and vertical spans
- Sprite drawing
- Screen composition via fixed-point perspective projection

**Game Logic** (`g_game.c`, `d_*.c`)
- Player state and input binding
- Game rules and progression (map transitions, episodes)
- Demo recording/playback
- Network synchronization (with game state consistency checks)
- Menu state machine

**UI & HUD** (`st_stuff.c`, `hu_stuff.c`, `am_map.c`)
- Status bar display
- Heads-up display (intermission, messages)
- Automap rendering

**Audio** (`s_sound.c`)
- Sound effect management and spatial audio
- Music playback

## Read These First

1. **linuxdoom-1.10/d_main.c** — Main game loop and initialization; explains the tick-based architecture
2. **linuxdoom-1.10/doomdef.h** — Core data structures and type definitions
3. **linuxdoom-1.10/g_game.c** — Game state machine and player state
4. **linuxdoom-1.10/p_setup.c** — Map loading and world initialization
5. **linuxdoom-1.10/r_main.c** — Rendering pipeline overview
6. **linuxdoom-1.10/Makefile** — Build configuration and module list
7. **linuxdoom-1.10/d_net.c** — Network protocol and sync mechanism
8. **README.TXT** — John Carmack's original release notes with architecture commentary
9. **sndserv/soundsrv.c** — Sound server architecture (if sound output is relevant)
