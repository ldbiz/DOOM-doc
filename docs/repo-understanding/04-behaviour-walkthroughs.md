# Behaviour Walkthroughs

This section documents the major behaviors supported by the engine, how they're triggered, and which parts of the code implement them.

## Behavior 1: Loading and Starting a Game Level

### Trigger

User selects a skill level and episode/map from the menu or via command-line options (`-skill`, `-warp`, `-episode`).

### Files Involved

- **Initialization**: `g_game.c::G_InitNew()`, `p_setup.c::P_SetupLevel()`
- **Map loading**: `p_setup.c::P_LoadThings()`, `p_setup.c::P_SpawnMapThing()`, `p_setup.c::P_CreateBlockMap()`
- **Player spawn**: `g_game.c::G_PlayerReborn()`, `p_mobj.c::P_SpawnMobj()`
- **State initialization**: `doomstat.h` (global state), `g_game.c` (player state)

### Bird's-Eye Flow

1. User confirms skill/map selection in menu (or game started from command line)
2. `G_InitNew()` is called with skill, episode, and map number
3. All mobjs in the world are destroyed (cleanup from any previous level)
4. New map data is loaded from WAD:
   - Map header and geometry (linedefs, vertices, sectors) parsed from WAD lump
   - Blockmap generated for efficient collision queries
   - BSP tree loaded (pre-built in WAD)
   - Map objects (things) spawned: player starts, monsters, pickups, decorations
5. Player is spawned at the player start position matching the current player number
6. Global state initialized: map timer set, kill/item/secret counts zeroed
7. Rendering system updated with new geometry
8. Game loop begins ticking the map

### Inputs

- Skill level (0–4)
- Episode number (1–4 for DOOM, 1 for DOOM2)
- Map number (1–9 for DOOM, 1–30 for DOOM2)

### Outputs

- Fully initialized map with:
  - All geometry loaded and indexed (sectors, linedefs, sectors)
  - BSP tree ready for rendering
  - All map objects spawned (player, monsters, items)
  - Thinker lists populated with initial object states
  - Blockmap built for collision queries

### External Calls and Side Effects

- **File I/O**: WAD file read via `W_CacheLumpName()`
- **Memory allocation**: Zone allocator used for map structures
- **Sound**: Optional startup sound may be triggered
- **Rendering**: Texture and sprite precaching via `R_PrecacheLevel()`

### Tests That Cover It

- Game can be started without crashing
- Player spawn position is correct
- Monster counts are correct
- Map geometry is visually accurate

---

## Behavior 2: Monster AI and Movement

### Trigger

Each game tick, every monster object's thinker is called via `P_Ticker()`.

### Files Involved

- **Core AI logic**: `p_enemy.c` (all AI decision-making)
- **Pathfinding**: `p_maputl.c::P_CheckPosition()`, `p_map.c::P_TryMove()`
- **Targeting**: `p_sight.c::P_CheckSight()`, `p_enemy.c::P_LookForPlayers()`
- **Damage**: `p_inter.c::P_DamageMobj()`
- **Movement**: `p_mobj.c::P_MobjThinker()` (applies velocity)
- **State machine**: `p_mobj.c::P_SetMobjState()`, `info.c` (state definitions)

### Bird's-Eye Flow

1. Monster's thinker is called each tick (function pointer from state machine)
2. **Idle/Wander Phase**: Monster looks for player:
   - `A_Look()` checks if player is visible
   - If player found, transitions to hunting state
3. **Chase Phase**: Monster moves toward player:
   - `A_Chase()` every tick
   - Calculate direction to player
   - Check for walls/obstacles between monster and player via `P_CheckPosition()`
   - Try to move toward player; pathfinding handles wall collision
   - If blocked, try moving left/right to strafe around obstacle
4. **Attack Phase** (if in range):
   - Melee attack: `A_Punch()`, `A_Claw()`, etc. trigger damage check on player
   - Ranged attack: Spawn projectile mobj with `P_SpawnMissile()`
   - Hitscan attack: Trace line from monster to player via `P_LineAttack()`
5. **State Transitions**:
   - Animations are state-based: chase → attack → chase → idle
   - Each frame/action defined in `info.c` state table

### Inputs

- Monster position, health, type
- Player position
- Map geometry

### Outputs

- Monster moves closer to player
- Monster may attack (projectile spawned or damage dealt)
- Monster animation updated
- Monster may die if health reaches 0

### External Calls and Side Effects

- **Collision checks**: Multiple `P_CheckPosition()` and `P_LineAttack()` calls
- **Vision checks**: `P_CheckSight()` traces line-of-sight
- **Projectile spawning**: `P_SpawnMissile()` creates new mobj for projectile
- **Damage**: `P_DamageMobj()` may be called on player or other monsters

### Tests That Cover It

- Monsters move toward player
- Monsters avoid walls (pathfinding works)
- Monsters can see through transparent sectors but not through walls
- Monsters attack when in range
- Monster death is handled correctly

---

## Behavior 3: Player Weapon Firing and Damage

### Trigger

Player presses fire button (typically spacebar or mouse click), processed by `G_BuildTiccmd()` and transmitted via network or directly applied.

### Files Involved

- **Input processing**: `g_game.c::G_BuildTiccmd()`
- **Weapon firing**: `p_pspr.c::P_FireWeapon()`, `p_pspr.c::P_CheckAmmo()`
- **Projectile spawning**: `p_mobj.c::P_SpawnMissile()`
- **Hitscan attacks**: `p_map.c::P_LineAttack()`, `p_map.c::P_AimLineAttack()`
- **Damage application**: `p_inter.c::P_DamageMobj()`
- **Animation**: `p_pspr.c` (weapon sprite state machine)

### Bird's-Eye Flow

**Hitscan Weapon (Pistol, Shotgun):**

1. Player input: fire button pressed; ticcmd includes `BT_ATTACK` flag
2. Check ammo: `P_CheckAmmo()` verifies player has ammo
3. Deduct ammo and trigger weapon firing animation (state change in weapon sprite)
4. Trace line from player view to maximum range:
   - `P_AimLineAttack()` traces line at player's aim angle
   - Intercepts all linedefs and mobjs in the way
5. Damage first hit target (monster or decoration):
   - `P_LineAttack()` calls `P_DamageMobj()` on first mobj hit
   - Damage amount depends on weapon
   - If damage reaches mobj's health ≤ 0, mobj dies
6. Sound effect triggered
7. Flash effect on screen and weapon sprite

**Projectile Weapon (Rocket, Plasma):**

1. Same ammo and input checks
2. Spawn new mobj for projectile:
   - `P_SpawnMissile()` creates projectile at weapon position
   - Projectile has thinker that moves it and checks for collisions
3. Projectile travels through map each tick
4. Projectile hits wall or mobj:
   - `P_CheckPosition()` detects collision
   - If mobj hit: `P_DamageMobj()` on target and nearby mobjs (splash damage for rockets)
   - Projectile is removed
5. Explosion effect triggered (sprites, sound)

### Inputs

- Fire button state (from keyboard/mouse input)
- Player position, angle, inventory (ammo, weapons)
- Map geometry and mobjs

### Outputs

- Ammo decremented
- Damage dealt to target
- Projectile spawned (for projectile weapons)
- Animation/sound/effect triggered
- Target may die

### External Calls and Side Effects

- **Collision/interception**: `P_AimLineAttack()` uses blockmap queries
- **Damage**: `P_DamageMobj()` may trigger target death, corpse physics, item spawning
- **Sound**: S_StartSound() triggered
- **Projectile physics**: Projectile thinker runs each tick, checking collision

### Tests That Cover It

- Hitscan hits correct target
- Projectile trajectory is correct
- Damage amounts are correct
- Ammo is consumed
- Out-of-ammo prevents firing
- Splash damage affects nearby targets
- Dead monsters spawn items

---

## Behavior 4: Sector Special Effects (Doors, Platforms, Damaging Floors)

### Trigger

Player or monster crosses a linedef with a special action, or manually triggers via switch/automap.

### Files Involved

- **Sector effects**: `p_spec.c` (door, floor, ceiling logic)
- **Activation**: `p_mobj.c::P_UseLines()`, `p_pspr.c::P_CheckAmmo()` (switch activation)
- **Tickers**: `p_doors.c`, `p_floor.c`, `p_ceilng.c` (per-tick state updates)
- **Damage application**: `p_inter.c::P_SectorDamage()`
- **State persistence**: `doomstat.c` (active sectors list)

### Bird's-Eye Flow

**Door Activation:**

1. Player or monster crosses linedef or uses switch
2. Linedef specifies door type (open, close, lock, etc.)
3. `EV_DoDoor()` is called:
   - Create door state machine (open, stay open, close, repeat)
   - Set initial state
   - Add to active sector effects list
4. Each tick, door thinker updates position:
   - `T_VerticalDoor()` moves ceiling toward target height
   - Check for obstruction (mobj in the way)
   - Sound effect as door moves
5. Door reaches target height or is blocked:
   - Transition to next state (stay open, close, reopen)
   - Sound effect when done

**Damaging Floor:**

1. Player enters sector with `DAMAGE_*` special type
2. Each tick in `P_Ticker()`, check if player in damaging sector
3. `P_DamageMobj()` called with damage amount per tick
4. Damage occurs every few ticks (not every tick)
5. Player death triggers if health ≤ 0

**Platform (Moving Floor/Ceiling):**

1. Activation similar to door
2. `EV_DoFloor()` or `EV_DoCeiling()` creates platform state machine
3. Platform moves at constant speed toward target height
4. Mobj on platform is carried along (position updated)
5. Mobj can be crushed if platform moves toward ceiling (damage)
6. Platform reaches target or is blocked:
   - Transition to next state (reverse direction, stop, etc.)

### Inputs

- Sector type and special flags
- Player/mobj position
- Activation trigger (crossing linedef, pressing switch)

### Outputs

- Sector geometry changes (floor/ceiling height)
- Mobjs on sector are moved (or crushed)
- Damage applied to mobjs in damaging sectors
- Sound/lighting effects

### External Calls and Side Effects

- **Movement**: `P_TryMove()` updates mobj position if on platform
- **Damage**: `P_DamageMobj()` called on mobjs in damaging sectors
- **State persistence**: Sectors added to active effects list, persisting across ticks
- **Sound**: `S_StartSound()` triggered for door/platform movement

### Tests That Cover It

- Door opens when activated
- Door closes correctly
- Door can be blocked by mobj
- Platform movement is smooth
- Damaging floor inflicts correct damage
- Crushing behavior works

---

## Behavior 5: Network Multiplayer Synchronization

### Trigger

Game started with `-net` flag and multiple node addresses specified.

### Files Involved

- **Network protocol**: `d_net.c` (tick-based sync, packet transmission)
- **Network I/O**: `i_net.c` (socket operations)
- **Player input**: `g_game.c::G_BuildTiccmd()`
- **Consistency checks**: `d_net.c::D_CheckNetGame()`, `g_game.c` (consistency counters)
- **State transmission**: `doomdata_t` structure (network packet format)

### Bird's-Eye Flow

1. **Initialization**: `D_CheckNetGame()` at startup:
   - Parse command-line node addresses
   - Create UDP sockets
   - Broadcast presence on network to discover other nodes
   - Wait for all players to acknowledge
2. **Per-Tick Synchronization**:
   - Each node builds its local player's ticcmd (input for the tick)
   - Node sends ticcmd to all other nodes via UDP
   - Node waits until it has received ticcmds from all other nodes for that tick
   - Once all received (or timeout), advance tick
3. **Tick Advancement**:
   - All nodes execute the same tick with same inputs → deterministic result
   - All nodes use same RNG seed (verified via consistency counter)
4. **Packet Format** (`doomdata_t`):
   - Header: sender node ID, packet sequence
   - Player ticcmds for up to `BACKUPTICS` ticks (buffer for reordering/retransmission)
   - Checksum for consistency verification
5. **Retransmission**:
   - If a node doesn't receive a remote node's ticcmd in time, it requests retransmission
   - Remote node resends its `BACKUPTICS` backlog
6. **Game Continues**:
   - If a node drops out, others detect missing ticcmds and may disconnect or wait

### Inputs

- Network configuration (node addresses)
- Local player ticcmd
- Ticcmds received from remote nodes

### Outputs

- All nodes maintain identical game state (deterministic tick-by-tick)
- Remote player inputs applied to their mobj
- Game progresses in lock-step

### External Calls and Side Effects

- **Network I/O**: UDP sockets used to send/receive ticcmds
- **Timing**: Real-time network delays must be handled (buffering, retransmission)
- **Determinism**: RNG and fixed-point math ensure same results on all nodes
- **Fallback**: If network is too slow, game may stall waiting for missing packets

### Tests That Cover It

- Single node (local game) runs normally
- Multi-node game starts and synchronizes
- All nodes reach same game state each tick
- Dropped packets are retransmitted
- Game continues even with network delays (up to timeout)
- Consistency check detects out-of-sync nodes
