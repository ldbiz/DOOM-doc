# BSP caveats and unknowns

## Missing repository context

- No WAD, map fixture, node-builder output, screenshots, or recorded expected traversal results are committed.
- No automated test harness is present, so behavior cannot be checked against small controlled BSP trees from this repository alone.
- The repository contains the runtime consumer but no BSP builder. The exact external node-builder/version used for intended maps is not documented here.
- The development paths referenced by `-wart` are external and unavailable.

## Format and compatibility questions

- The loaders clearly target classic Doom-format nodes, subsectors, segs, and map-lump ordering. Support for extended node formats, GL nodes, compressed nodes, Hexen-format maps, or wider indexes is not present or documented.
- Child identifiers are `unsigned short`, while traversal functions accept `int` and contain a special `bspnum == -1` case. The precise historical inputs that rely on this special case are not documented in the repo.
- `P_GroupLines` assumes every subsector has at least one valid seg and takes its sector from the first seg. No validation or alternative behavior is documented for malformed/empty leaves.
- The expected behavior for malformed node child indexes, seg indexes, inconsistent sidedefs, short/odd lump lengths, or undersized REJECT/BLOCKMAP lumps is not documented.

## Ambiguous or noteworthy implementation details

- `R_PointInSubsector` lives in renderer code but is a shared gameplay spatial primitive. A maintainer should clarify whether this ownership is intentional architecture or historical placement.
- `P_DivlineSide` contains a horizontal-line equality check using `x == node->y`; whether this is a preserved historical typo or relied-upon behavior is not explained by tests.
- The relationship between a subsector and exactly one sector is assumed and established from its first seg rather than encoded in the `SSECTORS` lump.
- Renderer child bounding boxes are trusted as supplied by the external node builder. The repo does not verify that they enclose their child geometry.
- The source comments say `REJECT` could act as a PVS, but this implementation visibly uses it for gameplay sight rejection, not renderer subtree visibility.

## Behavior visible in code but not covered by tests

- Front-to-back render traversal and solid-range-based bounding-box pruning.
- Side classification on partition lines, including points exactly on a line.
- Single-subsector/no-node maps and the special leaf-zero path.
- Sight traversal across both sides of a partition and slope narrowing through two-sided openings.
- REJECT bit indexing and interaction with objects' current subsector sectors.
- Seg clipping differences among one-sided walls, closed doors, windows, and visually empty two-sided lines.
- Fixed-capacity behavior for drawsegs, visplanes, openings, and other renderer buffers.
- WAD reload/override behavior when BSP-related map lumps change.

## BSP-adjacent distinctions to preserve

- BLOCKMAP is not the BSP. It is loaded from the same map group and shares linedefs, but it drives broad collision/path iteration.
- REJECT is not the BSP. It is a sector-pair early-rejection table used before detailed BSP sight traversal.
- Drawsegs and visplanes are not persistent BSP data. They are per-frame rendering products generated while processing visible leaves.
- The automap uses map lines and mapping flags; it is not evidence of a BSP visualization facility.

## Questions for a human maintainer

1. Which external WADs and node-builder outputs define the compatibility target for this source tree?
2. Should classic vanilla limits and malformed-map behavior be preserved exactly, or is safer rejection expected in future work?
3. Is the `P_DivlineSide` horizontal equality expression intentional historical compatibility behavior?
4. Are single-subsector maps and the `-1` child special case part of supported input, and can a minimal fixture be provided?
5. Is any external/manual regression process used to validate render traversal, clipping, sight, or BLOCKMAP behavior?
6. If BSP debugging is desired, should it be integrated into the automap, renderer instrumentation, or an external map-analysis tool?
