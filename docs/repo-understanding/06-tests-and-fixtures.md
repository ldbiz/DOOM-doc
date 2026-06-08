# Tests and Fixtures

## Test Framework

This codebase was written in 1993–1996, before modern unit testing practices were standard. **There are no unit tests, integration tests, or test fixtures in this repository.**

Verification is done through:
1. **Manual play-testing**: Running the executable and checking for visual artifacts, crashes, or gameplay errors.
2. **Original map compatibility**: Maps from the original DOOM game are included (in separate WAD files) and serve as de facto test cases.
3. **Code inspection**: Developers verify correctness by reading code and tracing execution logic.

## No Test Files

There are no files ending in `_test.c`, `_test.h`, `*_spec.c`, or similar. The entire codebase is production code with no test suite.

---

## Implicit Test Cases (Maps)

Although not distributed with this repository, the original DOOM game includes several maps that effectively serve as BSP test cases:

**DOOM I Maps**:
- **E1M1** ("Hangar") — Simple rectangular map; straightforward BSP.
- **E1M2** ("Nuclear Plant") — More complex geometry; multiple subsectors.
- **E1M3** ("Toxin Refinery") — Non-orthogonal geometry; diagonal walls and BSP splits.
- **E1M4** — **Iconic test for BSP correctness**: Contains tight corners, T-junctions, and complex partition lines.

**DOOM II Maps**:
- **MAP01–MAP32** — Increasingly complex geometry; later maps push BSP tree depth and segment counts.

To run tests manually:
1. Obtain original DOOM WAD files (doom.wad, doom2.wad) or free alternatives (freedoom.wad).
2. Run the executable with the WAD file:
   ```bash
   ./linuxdoom -iwad doom.wad -warp 1 1  # Load E1M1 (DOOM I)
   ./linuxdoom -iwad doom2.wad -warp 1   # Load MAP01 (DOOM II)
   ```
3. Observe rendering for visual correctness: no clipping artifacts, correct wall order, no floating sprites.

---

## BSP Behaviors Covered by Implicit Tests

### Map Loading & BSP Initialization
Every time a map loads, the entire BSP tree must be deserialized and wired correctly. Maps with different geometric complexity test various BSP tree depths and node counts.

### BSP Traversal Order
If the traversal order is incorrect, rendering produces wrong visual order (overdraw, sprites behind walls, etc.). Each map tests this implicitly by rendering correctly or not.

### Segment Clipping
Maps with walls that cross the screen edge or partially occlude others test the clipping logic. Incorrect clipping results in:
- Visible artifacts (partially drawn walls)
- Overdraw (sky showing through walls)
- Sprites clipped incorrectly

### Bounding Box Culling
Maps with large open areas or distant geometry test the bounding box visibility checks. If bounding boxes are wrong, either too much or too little geometry renders.

### Collision & Blockmap
Player movement tests collision detection. Maps with tight corridors, doorways, and obstacles test blockmap accuracy.

### Sight Checks
Enemy AI uses line-of-sight checks (p_sight.c) to determine if they can see the player. Incorrect sight checks result in:
- Monsters seeing through walls
- Monsters not engaging when they should
- Monsters ignoring the player when directly visible

---

## Known Coverage Gaps

The following BSP-related behaviors are *not* explicitly tested and are only verified through manual play-testing:

1. **Extreme BSP Tree Depths**: Very deep or unbalanced BSP trees may cause stack overflow or performance issues. Not tested.
2. **Pathological Geometry**: Maps with many thin wedges or perpendicular splits that create worst-case clipping scenarios. Only covered if maps happen to include them.
3. **Fixed-Point Overflow**: Extreme coordinate values or unusual partition lines could cause fixed-point overflow. Not tested.
4. **Maximum Limits**: Reaching MAXDRAWSEGS (256) or MAXSEGS (32) limits. Depends on map complexity.
5. **Multi-Player Spatial Queries**: Interaction between multiple players and BSP-based collision. Limited testing.

---

## Functional Correctness Indicators

Without formal tests, the following are used to verify BSP correctness:

- **Frame Rate**: If BSP traversal is correct and efficient, frame rate is stable. Poor performance may indicate inefficient traversal or excess clipping.
- **Visual Correctness**: Maps render with correct back-to-front order, no sprites floating behind walls, no clipping artifacts.
- **Collision**: Player/object movement is solid; no clipping through walls.
- **AI Behavior**: Monsters navigate correctly, engage with player when visible.
- **Demo Playback**: Recorded demo files that rely on deterministic physics and rendering.

---

## Running the Codebase Manually

To verify BSP behavior:

```bash
cd linuxdoom-1.10
make              # or manually compile with gcc
./linuxdoom       # With original DOOM WAD (doom.wad)
```

**In-game**:
- `-` / `+` keys: Cycle HUD size (affects rendering frustum)
- `F11` or `F12`: Toggle fullscreen (may affect clipping)
- `~` key: Open console (not enabled in all ports)
- `-devparm` flag (at startup): Enables developer mode with additional output

**Example command**:
```bash
./linuxdoom -iwad doom.wad -warp 1 1 -devparm
```

---

## Known Issues (From Code Comments)

Examining the code reveals a few acknowledged weaknesses in BSP handling:

**From `README.TXT`** (Author: John Carmack):

> The rendering proceded from walls to floors to sprites could be collapsed into a single front-to-back walk of the bsp tree to collect information, then draw all the contents of a subsector on the way back up the tree.

This suggests the developers recognized that the rendering could be more efficient by separating collection (traversal) from drawing (output) phases.

> The movement and line of sight checking against the lines is one of the bigger misses that I look back on. It is messy code that had some failure cases, and there was a vastly simpler (and faster) solution sitting in front of my face. I used the BSP tree for rendering things, but I didn't realize at the time that it could also be used for environment testing.

This indicates that BSP could have been used for collision and sight checks but wasn't—blockmap and explicit line tests are used instead.

---

## Fixture Data

There are no BSP-specific fixture files in this repository (e.g., minimal maps for testing). All fixtures are implicit in the map lumps within WAD files, which are *not* included here.

To create a test fixture:
1. Use a map editor (DeuTecH, WadAuthor, Slade3, TrenchBroom) to create a minimal map with specific BSP characteristics.
2. Add the map lump to a WAD file.
3. Load the WAD with the engine and verify correctness.

---

## Summary

| Category | Status | Notes |
|----------|--------|-------|
| Unit Tests | ❌ None | No test framework |
| Integration Tests | ❌ None | Manual play-testing only |
| Test Fixtures | ❌ None | Original maps serve as implicit fixtures |
| BSP Coverage | ✅ Implicit | Every map loads and renders; all BSP behaviors covered in practice |
| Known Gaps | ⚠️ Yes | Extreme depths, pathological geometry, overflow scenarios |
| Manual Testing | ✅ Yes | Play-testing is the primary verification method |
