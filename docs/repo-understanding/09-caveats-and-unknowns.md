# Caveats and Unknowns

This section documents points of uncertainty, unclear behavior, and missing context that a human maintainer should be aware of.

---

## Architecture and Design

### BSP Tree Pre-Building

**Uncertain**: Whether BSP trees can be rebuilt at runtime or must be pre-built by external tools.

**Evidence**: 
- `p_setup.c::P_LoadNodes()` loads BSP tree from WAD file
- No code in repository rebuilds or validates BSP tree
- Comments suggest BSP was built by map editors before release

**Question**: If a map geometry is modified at runtime, can the BSP be regenerated, or must a new map be loaded?

**Implication**: Complex or impossible to add dynamic geometry changes (e.g., destructible walls, new passages).

---

### Rendering: Polar Coordinates vs Fixed-Point

**Noted**: The README mentions "using polar coordinates for clipping comes to mind" as a design decision that the author (Carmack) now considers questionable.

**Uncertain**: Whether the current Linux release still uses polar coordinates or has been refactored to fixed-point.

**Evidence**: Code in `r_segs.c` and `r_things.c` uses fixed-point (`fixed_t`) for calculations.

**Implication**: Historical note; the actual implementation may be better than the README suggests. Verify before proposing large rendering refactors.

---

### Monster Pathfinding Algorithm

**Uncertain**: Exact algorithm used for monster pathfinding when blocked by walls.

**Evidence**: `p_enemy.c::A_Chase()` attempts to move left/right when blocked forward.

**Incomplete Understanding**:
- Does it use a proper A* or Dijkstra algorithm?
- Does it retry every few ticks or give up?
- Can monsters get permanently stuck in corners?

**Question**: What happens if a monster is completely surrounded and cannot reach the player?

**Implication**: Unknown performance characteristics or potential infinite loops if pathfinding logic is modified.

---

## Platform Dependencies and Compatibility

### X11 Graphics Backend

**Uncertain**: Full compatibility matrix across different X11 implementations.

**Known**: 
- Code written for Unix X11 servers
- Linked against libX11, libXext

**Unknown**:
- Does it work on modern Wayland systems?
- Behavior on headless servers or remote X11
- Behavior with non-8-bit-per-pixel displays

**Question**: Why is the game hard-coded to 320x200 resolution? Is this a limitation or a design choice?

**Implication**: Porting to modern systems may require significant display system refactoring.

---

### Audio Subsystem

**Uncertain**: Exact behavior when sound server is unavailable.

**Evidence**: `s_sound.c::S_Init()` can initialize sound as a stub (no output) if conditions aren't met.

**Questions**:
- What triggers stub mode vs real sound mode?
- Does game crash or silently continue with no sound?
- What happens if `sndserv` process crashes after game start?

**Implication**: Audio robustness is unclear; could be fragile in edge cases.

---

### Network Protocol: UDP Port and Permissions

**Unclear**: Which UDP port the game uses and whether it requires elevated privileges.

**Evidence**: `i_net.c` defines `DOOMPORT = (IPPORT_USERRESERVED + 0x1d)` (a user-level port > 1024).

**Questions**:
- Is the port configurable?
- Can multiple DOOM instances run on the same machine?
- What happens if the port is already in use?

**Implication**: Local multiplayer setup and firewall configuration are unclear.

---

## Numerical Precision and Determinism

### Fixed-Point Arithmetic Limits

**Uncertain**: Precision implications of 16.16 fixed-point arithmetic on modern CPUs.

**Evidence**: `m_fixed.h` defines `fixed_t` as a 32-bit integer with 16 bits of fractional precision.

**Potential Issues**:
- Very large map coordinates could overflow
- Very small movements could be lost to quantization
- Network sync depends on deterministic rounding

**Question**: Are there any known precision issues or edge cases in the current maps/game?

**Implication**: Adding new maps or physics could expose precision bugs.

---

### Consistency Counters vs Actual State

**Uncertain**: Whether the consistency counter in network code truly validates game state equality.

**Evidence**: `g_game.c::consistancy[][]` stores a CRC-like value each tick.

**Questions**:
- Is it a true checksum or just a simplified hash?
- Could two different game states produce the same checksum?
- Is the algorithm adequate for detecting network desync?

**Implication**: Network multiplayer could have subtle bugs that the consistency check doesn't catch.

---

## AI and Behavior

### Monster Decision-Making Randomness

**Uncertain**: How randomness affects determinism in networked games.

**Evidence**: `m_random.c` defines deterministic RNG (same seed = same sequence).

**Questions**:
- Is RNG state initialized identically on all networked nodes?
- Could RNG state diverge due to timing differences?
- How often do ties occur in AI decision-making?

**Implication**: Network desync could theoretically occur if RNG handling has a bug.

---

### Monster Melee Attack Range

**Unclear**: Exact melee attack range and hitbox calculations.

**Evidence**: `p_enemy.c` has various attack functions (punch, claw, bite, etc.).

**Questions**:
- Is range calculated from center or edge of sprite?
- Do hitboxes perfectly match sprite boundaries?
- Are there range check edge cases?

**Implication**: Monster balance and difficulty perception depend on unclear attack ranges.

---

### Line-of-Sight Calculation

**Uncertain**: Whether `p_sight.c::P_CheckSight()` perfectly matches monster visibility.

**Evidence**: Complex 2D line-clipping algorithm in sight checking.

**Potential Issues**:
- Floating-point precision in line-of-sight tests
- Edge cases at sector boundaries
- Interaction with BSP tree traversal

**Question**: Have players reported monsters "seeing through walls"?

**Implication**: Monster AI behavior could seem unfair if sight checking has subtle bugs.

---

## Game Flow and Progression

### Demo Recording Format Compatibility

**Uncertain**: Whether demo format has changed between versions or DOOM vs DOOM2.

**Evidence**: `g_game.c` saves raw ticcmd data.

**Questions**:
- Are old DOOM1 demos compatible with this DOOM2 build?
- What happens if you try to play a demo from incompatible version?
- Is there version checking in demo loading?

**Implication**: Old demo files could silently desync or crash if version checking is absent.

---

### Save Game Format

**Uncertain**: Exact save game compatibility and corruption handling.

**Evidence**: `p_saveg.c` serializes game state to disk.

**Questions**:
- Is there version information in save files?
- What happens if you load a save from different DOOM version?
- Behavior on corrupted/truncated save files?

**Implication**: Save game data could be lost silently if corruption detection is weak.

---

### Episode/Map Progression Rules

**Uncertain**: Exact rules for unlocking episodes or secret maps.

**Evidence**: DOOM I has multiple episodes; some levels are secret.

**Questions**:
- Are episodes progressively unlocked or all available?
- How are secret levels reached (exit lines, keys)?
- Behavior if player tries to warp to unavailable level?

**Implication**: Game design intent unclear; could be misunderstood by modders.

---

## Code Quality and Testing

### Memory Leak Potential

**Uncertain**: Whether zone allocator properly frees all memory.

**Evidence**: Zone allocator uses simple bump-pointer with no true freeing.

**Questions**:
- Are there memory leaks on map transitions?
- Long multiplayer sessions: does memory grow unbounded?
- Savegame restoration: are old pointers cleaned up?

**Implication**: Game could crash or slow down after many hours of play.

---

### Integer Overflow Risks

**Uncertain**: Whether any counters could overflow.

**Evidence**: `gametic` is a 32-bit int (can overflow in ~24 days of continuous play).

**Questions**:
- How does overflow behave (wrapping vs crash)?
- Are there safeguards for long play sessions?
- Interaction with network protocol (packet numbering)?

**Implication**: Long play sessions or stress tests could trigger unexpected behavior.

---

### Pointer Dereferences and Null Checks

**Uncertain**: Consistency of null pointer checking.

**Evidence**: Code mixes defensive null checks with unsafe dereferences.

**Questions**:
- Are there functions that assume non-null pointers without checking?
- Could invalid WAD data cause null pointer crashes?
- Are there guard rails for corrupted save games?

**Implication**: Robustness against corrupted/malicious input unclear.

---

## External Tools and Workflows

### Map Editor Used and Compatibility

**Uncertain**: Which map editor created the original DOOM maps.

**Evidence**: README mentions map editors and BSP trees.

**Questions**:
- Can modern map editors (WadAuthor, Slade3, etc.) produce compatible maps?
- Are there undocumented WAD lumps or features?
- Can maps be edited safely or are there version issues?

**Implication**: Creating new content requires tool knowledge not in repository.

---

### Sound Data Format

**Uncertain**: Exact audio format stored in WAD lumps.

**Evidence**: `sndserv/wadread.c` reads sound data from WAD.

**Questions**:
- Is it raw PCM or compressed?
- Sample rate and bit depth?
- Can non-standard WAD sound data be handled?

**Implication**: Modifying or adding sounds requires understanding WAD sound format (not documented in repo).

---

## Build and Development Environment

### Dependency Versions

**Uncertain**: Exact versions of X11 libraries required.

**Evidence**: Makefile links against libX11, libXext, but no version specified.

**Questions**:
- Minimum X11 version?
- Behavior with very old or very new X11?
- Other hidden dependencies (libc version, GCC version)?

**Implication**: Reproducible builds on different systems uncertain.

---

### Platform-Specific Code Paths

**Uncertain**: Coverage of different Unix variants.

**Evidence**: Code mentions Linux, but also references to Irix, BSD, etc.

**Questions**:
- Which Unix platforms are actually tested?
- Are there untested code paths that could fail?
- Compatibility with WSL, Cygwin, or other compatibility layers?

**Implication**: Porting to new platforms could surface latent bugs.

---

## Known Issues in README

The README.TXT lists architectural regrets but doesn't provide workarounds:

1. **Polar coordinates for clipping** — If still present, could be a performance issue
2. **Movement/line-of-sight code is messy** — Could contain bugs or inefficiencies
3. **BSP tree not used for environment testing** — Could be optimized

**Question**: Which of these historical issues remain in the current codebase?

**Implication**: Performance and correctness concerns should be investigated before large-scale changes.

---

## Inferred Behaviors Needing Confirmation

### Deathmatch Mode Details

**Observed**: `-deathmatch` flag mentioned in code; behavior inferred but not fully documented.

**Inferred**:
- No monsters spawn in DM mode
- Weapons respawn
- Friendly fire enabled
- Frags are goal instead of completing level

**Unconfirmed**: Exact game-end condition, scoring rules, or tie-breaking behavior.

**Question**: Is deathmatch fully working or partially implemented?

---

### Network Join/Leave Behavior

**Uncertain**: What happens when a networked player drops out mid-game.

**Evidence**: `d_net.c` tracks `nodeingame[]` flags.

**Questions**:
- Does game pause or continue for other players?
- Can player reconnect to game in progress?
- Save state preserved or game continues from current state?

**Implication**: Network multiplayer could be fragile if player dropout handling is incomplete.

---

## Observations Needing Clarification

### "linuxxdoom" Executable Name

**Observation**: Executable built as `linux/linuxxdoom` (two x's).

**Question**: Why the extra 'x'? Is it a typo or intentional naming?

**Implication**: Non-standard naming could confuse users expecting `linuxdoom`.

---

### CVS Directory in linuxdoom-1.10/

**Observation**: Directory contains `CVS/` folder (Concurrent Versions System metadata).

**Question**: Is this an older checkout that should be cleaned? Is VCS history important?

**Implication**: Repository structure could be simplified or metadata preserved for historical reasons.

---

## Summary of High-Priority Unknowns

These should be investigated by a human maintainer if the code will be significantly modified:

1. **Network robustness**: What happens if a player disconnects mid-game?
2. **Memory management**: Are there leaks over long play sessions?
3. **Precision limits**: What are the edge cases for fixed-point math?
4. **Audio behavior**: What happens if sound server unavailable?
5. **Map compatibility**: Can maps be edited with modern tools?
6. **Demo format**: Is demo compatibility maintained across versions?
7. **Platform support**: Which Unix platforms are actually tested?
8. **Determinism**: Is consistency check sufficient for network desync detection?

Addressing these would significantly reduce uncertainty about the codebase's reliability and portability.
