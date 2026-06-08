# Configuration and Environment

## Configuration Files

### DOOM.CFG (User Configuration)

**Location**: `$HOME/.doomrc` on Linux (varies by platform variant)

**Purpose**: Stores user preferences that persist across game sessions.

**Contents**:
- Key bindings (attack, use, move forward, etc.)
- Graphics mode and resolution
- Audio volume and settings
- Mouse sensitivity
- Game options (screen size, gamma correction, etc.)

**Format**: Binary configuration file (not human-readable)

**Loading**: Called from `D_DoomMain()` via `M_LoadDefaults()` in `m_misc.c`

**Saving**: Called on exit or when menu options changed via `M_SaveDefaults()`

**Key Variables** (from `m_misc.c`):
- `key_right`, `key_left`, `key_up`, `key_down` — movement keys
- `key_fire`, `key_use`, `key_strafe` — action keys
- `mousebfire`, `mousebstrafe` — mouse button bindings
- `sfxVolume`, `musicVolume` — audio levels
- `screenSize` — HUD size (0-10)
- `detailLevel` — graphics detail (high/low)
- `usegamma` — gamma correction level

---

## Environment Variables

The game respects standard Unix environment variables:

| Variable | Effect |
|----------|--------|
| `HOME` | Used to locate configuration file (`$HOME/.doomrc`) |
| `DISPLAY` | X11 display for graphics output (e.g., `:0`, `:0.0`) |
| `DOOMWADDIR` | Optional path to search for WAD files |

**Implicit Dependencies**:
- X11 server must be running and accessible via `$DISPLAY`
- Sound device accessible via `/dev/dsp` or similar (varies by platform)

---

## Command-Line Arguments

Parsed in `D_DoomMain()` and accessible throughout via global arrays `myargc`/`myargv`:

### Game Selection

```
-iwad <filename>         # IWAD file path (DOOM.WAD, DOOM2.WAD, etc.)
-file <files>            # Additional WAD files (PWAD) to load
```

**Defaults**: If `-iwad` not specified, searches for DOOM.WAD or DOOM2.WAD in:
- Current directory
- `$HOME`
- Standard install paths

**Purpose**: Specify which game to play and any mods/PWADs to load.

### Game Difficulty and Progression

```
-skill <0-4>             # Difficulty: 0=TNFTNT, 1=HNTR, 2=HMP, 3=UV, 4=Nightmare
-episode <1-4>           # DOOM I episode (1-4; ignored for DOOM2)
-warp <episode> <map>    # Start at specific level
```

**Defaults**: 
- Skill: 2 (Hurt Me Plenty)
- Episode: 1
- Map: 1

**Example**:
```bash
./linuxxdoom -iwad DOOM2.WAD -skill 3 -warp 2 5   # DOOM2, skill 3, map 05
```

### Network and Multiplayer

```
-net <node0> <node1> ... <nodeN>   # Network game
-deathmatch                          # DM mode
-altdeath                            # Alternate DM rules
-nomonsters                          # Disable monsters
-respawn                             # Monster respawning
-fast                                # Double monster speed
```

**Example**:
```bash
./linuxxdoom -net 192.168.1.10 192.168.1.20   # Network DM
```

### Demo and Recording

```
-record <demoname>      # Record a demo
-playback <demoname>    # Play back a demo
-timedemo <demoname>    # Play demo and report frame time
```

**Demo File**: Stored in current directory with `.lmp` extension

**Example**:
```bash
./linuxxdoom -record demo1           # Records demo1.lmp
./linuxxdoom -playback demo1         # Plays demo1.lmp
```

### Save Games

```
-save <slotnum>         # Load save game from slot (0-5)
```

**Save File Location**: `$HOME/DOOM_SAVEGAME<slotnum>` (varies by platform)

### Graphics and Display

```
-width <pixels>         # Screen width (usually 320)
-height <pixels>        # Screen height (usually 200)
-fullscreen             # Fullscreen mode (if supported)
-window                 # Windowed mode (if supported)
```

**Defaults**: 320x200 windowed

### Performance and Profiling

```
-nodraw                 # Skip rendering (performance test)
-noblit                 # Skip screen update (performance test)
-timingdemo <demoname>  # Play demo and measure FPS
```

### Development and Debugging

```
-devparm                # Development mode (some debug output)
-debug                  # Enable debug output
-noprompt               # Suppress prompts
-nomusic                # Disable music
-nosound                # Disable sound effects
```

### Response Files

```
@<filename>             # Load arguments from response file (one per line)
```

**Example**: `response.txt` contains `-iwad DOOM2.WAD -skill 3`

```bash
./linuxxdoom @response.txt
```

---

## Defaults and Required Settings

### Required to Run

- **IWAD file**: Must be findable (either via `-iwad` or in search paths)
  - DOOM.WAD (DOOM I)
  - DOOM2.WAD (DOOM II)
  - Or compatible iwad

- **X11 display**: Must be available (`$DISPLAY` set or `:0` available)

### Optional Configuration

- **WAD files**: Defaults to IWAD only; additional PWADs optional
- **Key bindings**: Defaults applied if not in `DOOM.CFG`
- **Audio**: Defaults to max volume; can be muted

### Defaults if Not Specified

| Setting | Default |
|---------|---------|
| Skill | 2 (Hurt Me Plenty) |
| Episode | 1 |
| Map | 1 |
| Graphics mode | 320x200 windowed |
| Audio volume | Max |
| Gamma correction | 0 (medium) |
| Detail level | High |
| Game mode | Single player |

---

## Runtime Consequences of Important Settings

### Skill Level

- **0 (I'm Too Young to Die)**: Monsters deal half damage, player gains 25% more health from pickups, easy difficulty monsters spawn in higher difficulties
- **1 (Hey, Not Too Rough)**: Monsters deal 75% damage, monster count reduced
- **2 (Hurt Me Plenty)**: Normal damage, normal monster count
- **3 (Ultra-Violence)**: Normal damage, all monsters spawn (including high-tier ones)
- **4 (Nightmare)**: Monsters regenerate 5 HP/sec, double speed, triple reaction time, respawn enabled

**Code**: Checked in `g_game.c` and `p_mobj.c` when spawning monsters and applying damage.

### Deathmatch Mode

When enabled:
- No monsters spawn (only player/respawn points)
- Weapons/ammo respawn indefinitely
- Player fragging is goal instead of monster killing
- Friendly fire enabled
- Weapons/power-ups distributed throughout map

**Code**: Checked throughout `g_game.c`, `p_setup.c`, `p_inter.c`

### Monster Respawning

When enabled:
- Dead monsters respawn after delay (8+ seconds)
- Respawn at original spawn location if available
- Enabled for skill Nightmare and with `-respawn`

**Code**: `p_mobj.c::P_MobjThinker()` checks respawn flag

### Fast Monsters

When enabled (skill Nightmare or `-fast`):
- Monster speed doubled
- Monster reaction time tripled
- Projectile speed increased

**Code**: Checked in `p_enemy.c` and projectile spawning code

### No Monsters Mode

When enabled:
- Player spawn points are created but monsters are not
- Pickups still spawn
- Game goal is to reach the exit (not to kill everything)

**Code**: Checked in `p_setup.c::P_SpawnMapThing()`

---

## Local/Dev/Test Examples

### Development Setup

```bash
# Build with debug symbols
make -C linuxdoom-1.10 CFLAGS="-g -Wall -DNORMALUNIX -DLINUX"

# Run with debug output
./linuxdoom-1.10/linux/linuxxdoom -devparm -iwad DOOM2.WAD

# Profile frame time
./linuxdoom-1.10/linux/linuxxdoom -timingdemo demo1
```

### Testing Demos

```bash
# Record a demo
./linuxdoom-1.10/linux/linuxxdoom -iwad DOOM.WAD -record test_demo

# Play back the demo
./linuxdoom-1.10/linux/linuxxdoom -iwad DOOM.WAD -playback test_demo

# Time demo playback (for performance regression testing)
./linuxdoom-1.10/linux/linuxxdoom -iwad DOOM.WAD -timedemo test_demo
```

### Local Multiplayer (Serial)

```bash
# Serial-based on two machines
./linuxxdoom -iwad DOOM.WAD -net
# (Would require serial port setup; not commonly used)
```

### Network Multiplayer

```bash
# Machine 1 (192.168.1.10)
./linuxxdoom -iwad DOOM.WAD -net 192.168.1.10 192.168.1.20

# Machine 2 (192.168.1.20)
./linuxxdoom -iwad DOOM.WAD -net 192.168.1.10 192.168.1.20
```

---

## Secrets and Credentials

This repository contains **no secrets or credentials**. It is pure game engine source code.

Game WAD files (IWAD files like DOOM.WAD, DOOM2.WAD) are not included and must be obtained separately from commercial game distributions or legitimate sources.

Network multiplayer uses:
- Standard UDP protocol (no authentication or encryption)
- No API keys or tokens
- No external service dependencies

---

## Sound Server Configuration

The optional `sndserv` (sound server) is automatically spawned by `S_Init()` in `s_sound.c` if:
- X11 audio device is detected
- Compilation supports sound output

**Environment for sndserv**:
- Inherits `$DISPLAY` from parent process
- Accesses sound device directly (typically `/dev/dsp` on older Linux)
- Uses shared memory or named pipes to communicate with main game process

**Configuration**: No explicit configuration file; behavior controlled by `sndserv` command-line arguments (if any) passed by `S_Init()`.
