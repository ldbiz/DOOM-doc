# Tests and Fixtures

## Test Framework

This repository does **not include automated tests**. It is the original source code release from id Software and predates modern unit testing practices.

Testing was performed manually during development:
- Gameplay testing (does the game play and not crash?)
- Network testing (do multiplayer games synchronize?)
- Demo recording/playback (are inputs reproducible?)
- Performance profiling (frame rate on target hardware)

The game includes some built-in testing facilities described below.

---

## How Tests Are (Could Be) Run

### Build Verification

```bash
cd linuxdoom-1.10
make clean
make                    # Should compile without errors
```

This is the primary "test" — if the code compiles and links, the build succeeds.

### Runtime Smoke Tests

```bash
# Check if the game runs without crashing
./linux/linuxxdoom -iwad DOOM.WAD -warp 1 1 &
sleep 2
kill %1          # Game should not crash during startup

# Check demo playback works
./linux/linuxxdoom -iwad DOOM.WAD -timedemo demo1  # Should complete
```

### Demo-Based Testing

The game's demo recording/playback mechanism serves as a form of regression testing:

```bash
# Record a playthrough
./linux/linuxxdoom -iwad DOOM.WAD -record mytest

# Playback should reproduce identical gameplay
./linux/linuxxdoom -iwad DOOM.WAD -playback mytest

# Timed playback for performance regression
./linux/linuxxdoom -iwad DOOM.WAD -timedemo mytest
```

If demo playback produces different results (desyncs), it indicates:
- Determinism broken (RNG or fixed-point math issues)
- Network sync problem (if network-based demo)
- Platform-specific issue (floating-point differences, etc.)

### Performance Profiling

```bash
# No rendering (pure game logic performance)
./linux/linuxxdoom -iwad DOOM.WAD -nodraw -timedemo mytest

# No screen update (rendering performance)
./linux/linuxxdoom -iwad DOOM.WAD -noblit -timedemo mytest

# Full game performance
./linux/linuxxdoom -iwad DOOM.WAD -timedemo mytest
```

---

## Important Test-Related Files

| File | Purpose |
|------|---------|
| `linuxdoom-1.10/ChangeLog` | Records bugs fixed and features tested |
| `linuxdoom-1.10/TODO` | Known issues and planned improvements |
| `linuxdoom-1.10/FILES` | Directory listing (historical documentation) |
| `linuxdoom-1.10/README.asm` | Assembly optimization notes |
| `linuxdoom-1.10/README.book` | References for algorithm validation |
| `sndserv/README.sndserv` | Sound server testing notes |

---

## Behaviors Covered by Built-In Testing

### Game State Consistency (Implicit)

The game's internal consistency checks validate:

- **Demo Recording/Playback**: If a recorded demo plays back identically, the engine is deterministic
  - Triggered in `g_game.c::G_CheckDemoStatus()`
  - Consistency counter in `g_game.c::consistancy[][]` checks RNG state matches across network nodes

- **Network Sync**: If networked games complete without desyncing, synchronization is correct
  - Monitored in `d_net.c::D_NetCmd()` and `g_game.c::G_Ticker()`
  - Checksum verification in `d_net.c::NetbufferChecksum()`

### Stress Testing

- **Long gameplay sessions**: Game can run indefinitely without memory leaks
- **Full map traversal**: Player can reach all areas without getting stuck
- **Monster pathfinding**: Monsters can navigate all map layouts
- **Collision detection**: No clips through walls or unexpected collisions

### Platform Testing

- **X11 Graphics**: Rendering on Unix/Linux X11 servers
- **Audio Output**: Sound playback on various Unix audio systems
- **Network I/O**: UDP socket operations across networks
- **File I/O**: WAD file loading on various filesystems

---

## Fixtures and Sample Data

### WAD Files

WAD files are the primary "fixtures" for testing. They contain:

| Type | Source | Purpose |
|------|--------|---------|
| **IWAD** | Official game distribution | Complete game data (maps, sprites, sounds, textures) |
| **PWAD** | Mods or custom content | Test map modifications, custom objects |
| **Demo Files** | Recorded gameplay | Determinism and regression testing |

**No sample WADs are included** in this repository; they must be obtained separately.

### Built-In Test Maps

The game includes maps for various purposes:

- **Episode 1, Map 1**: Tutorial-like level with basic enemies and items
- **Episode 1, Map 9**: Secret level (accessed via hidden exit)
- **Deathmatch spawns**: Maps include deathmatch spawn points for multiplayer testing

### Test Scenarios (Implicit in Map Design)

The official maps implicitly test:

- **Rendering**: Each map exercises different rendering paths (tall ceilings, dark sectors, many sprites, etc.)
- **Physics**: Maps include doors, platforms, damaging floors for mechanic testing
- **AI**: Monster placement tests pathfinding, targeting, and decision-making
- **Performance**: Various map complexity levels (sparse to dense)

---

## Obvious Coverage Gaps

The original release does not include:

1. **Unit Tests**: No isolated testing of individual functions
   - *Gap*: Logic bugs in utility functions (math, memory) would not be caught
   - *Example*: `P_CheckPosition()` collision logic has no unit tests

2. **Edge Case Testing**: No systematic testing of boundary conditions
   - *Gap*: Monsters at map edges, extreme coordinates, zero-health scenarios
   - *Example*: Teleport to invalid location could crash

3. **Regression Tests**: No automated suite of tests to verify fixes don't rebreak issues
   - *Gap*: Fixed bugs could resurface if code is modified
   - *Example*: Crashing bug in monster AI might reoccur if `p_enemy.c` is heavily refactored

4. **Memory Safety Tests**: No tools like AddressSanitizer or Valgrind runs documented
   - *Gap*: Memory leaks, buffer overflows, use-after-free bugs would not be detected
   - *Example*: Zone allocator has no memory safety checks

5. **Network Stress Tests**: No documented testing of network lag, packet loss, or node dropouts
   - *Gap*: Network synchronization issues under realistic conditions (high latency, dropped packets)
   - *Example*: Behavior when a node disconnects mid-game is unclear

6. **Platform-Specific Tests**: No documented testing on various Unix variants
   - *Gap*: Issues on non-Linux systems not caught
   - *Example*: Endianness issues, X11 variant compatibility, sound device differences

7. **Performance Regression Tests**: No automated benchmarking
   - *Gap*: Optimizations could be accidentally reversed
   - *Example*: If BSP traversal is made less efficient, no test would catch it

---

## Inferred Behaviors from Code (Not Explicitly Tested)

### AI Pathfinding Edge Cases

Code in `p_enemy.c::A_Chase()` attempts to handle:
- Monster stuck on wall (tries left/right strafe)
- Monster blocked by other monster (gives up)
- Monster separated from player by lava (can't cross)

*Verification*: Not explicitly tested; inferred from code comments and structure.

### Network Packet Retransmission

Code in `d_net.c` implements retransmission logic:
- Track which ticks each node has acknowledged
- Request retransmission if a tick's ticcmd not received in time

*Verification*: Tested implicitly by gameplay; no explicit test harness.

### Sound Server Startup

Code in `s_sound.c::S_Init()` spawns `sndserv`:
- Fork and exec `sndserv` binary
- Set up communication pipe/shared memory

*Verification*: Works if sound plays; fails if no audio. No explicit error handling tested.

### Save Game Corruption Recovery

Code in `p_saveg.c` reads save files:
- Validation of magic number
- Version checking
- Incomplete save detection

*Verification*: Inferred from code; behavior on corrupted saves not documented.

---

## Integration Testing (Implicit)

The game's primary integration test is a full playthrough:

1. Start game
2. Play through a map
3. Complete level
4. Proceed to next level
5. Play through all maps
6. Reach game end

If all these steps complete without crashing or desyncing, integration is working. This is the "smoke test" that the original developers relied upon.

### Example Command for Integration Test

```bash
# Single-player playthrough (would need user input, so not fully automated)
./linux/linuxxdoom -iwad DOOM2.WAD -skill 2 -warp 1 1

# Demo-based integration test (automated)
./linux/linuxxdoom -iwad DOOM2.WAD -playback longplay_demo
```

If the demo plays to completion without desyncing, integration is verified.
