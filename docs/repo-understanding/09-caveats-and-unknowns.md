# 09 - Caveats & Unknowns

This file captures BSP-related points that could not be confirmed from the repo alone, ambiguous naming, and areas where external context or maintainer knowledge would be needed.

## Missing context

### BSP node builder

The DOOM engine **consumes** BSP trees but does **not build** them. The BSP builder is a separate tool (id Software's internal `doombsp` or community tools like `BSP`, `ZDBSP`, `ZokumBSP`). This repo contains:
- No BSP generation code.
- No specification of the exact partitioning algorithm.
- No documentation of the BSP builder's splitting heuristics or seg-splitting rules.

**Implication**: If you need to understand *why* the BSP tree has a particular topology, or how segs are split at partition lines, you must look outside this repo.

### Coordinate system and precision

The partition line side test (`R_PointOnSide()` in `r_main.c`) uses fixed-point math with various optimizations (sign-bit checks, early returns for axis-aligned cases). The exact numerical behaviour — especially edge cases where a point lies exactly on a partition line — is not documented.

### REJECT table format

The REJECT lump is loaded as a raw byte array. Its exact bit-packing format (which bit corresponds to which sector pair) is assumed to follow the standard DOOM convention (`pnum = s1 * numsectors + s2; bytenum = pnum >> 3; bitnum = 1 << (pnum & 7)`), but:
- No validation checks that the lump size matches `ceil(numsectors * numsectors / 8)`.
- No fallback behaviour if the lump is missing or truncated.
- The REJECT table is built by the same external node-builder tool.

## Ambiguous naming

| Term | Ambiguity |
|---|---|
| `firstline` in `subsector_t` | Despite being called `firstline`, it indexes into `segs[]`, not `lines[]`. The field name suggests linedefs but it actually indexes segs. |
| `numlines` in `subsector_t` | Same issue — refers to seg count, not line count. |
| `children[]` in `node_t` | Holds `unsigned short` values that can be either node indices or subsector indices (with `NF_SUBSECTOR` bit). The type gives no clue about this dual use. |
| `side` return from `R_PointOnSide()` | Returns 0 for "front" and 1 for "back", but the geometric meaning of front/back depends on the partition line direction. The code uses these directly as array indices. |
| `solidsegs` | In this port, the type is `cliprange_t`, not `solidseg_t` as in some other ports. The variable is named `solidsegs` but declared as `cliprange_t solidsegs[MAXSEGS]`. |

## Behaviour inferred but not clearly visible

### Visplane merging algorithm

`R_CheckPlane()` decides whether a new X range can be appended to an existing visplane by checking if `top[]` entries in the overlap are `0xff` (unset). The exact sentinel value and the logic for creating new visplanes when ranges don't merge cleanly is spread across `R_FindPlane()` and `R_CheckPlane()` — the decision tree is not documented.

### Sprite clipping interaction with drawsegs

`R_DrawSprite()` walks drawsegs backwards (from `ds_p-1` to `drawsegs`) building per-column clip arrays. The exact rules for how `silhouette`, `bsilheight`, `tsilheight`, `sprtopclip`, and `sprbottomclip` interact are complex and only visible by reading the code line-by-line.

### `-1` subsector edge case

In `r_bsp.c`, `R_RenderBSPNode()` has a special case: `if (bspnum == -1) R_Subsector(0)`. This handles single-subsector maps where `numnodes == 0`. The root traversal call passes `numnodes - 1` which would be `-1`. This edge case is not documented in comments.

## External references not present in the repo

- **DOOM Specs / Unofficial DOOM Specs (UDS)**: Documents the WAD format and lump layouts. Not included in this repo.
- **DOOM BSP FAQ**: Community documentation on BSP tree construction and traversal. Not included.
- **Original DOS source code**: The Linux port was adapted from DOS; the original DOS renderer and sound code are not present (copyrighted sound library prevented release).
- **Microsoft Windows port**: Mentioned in `README.TXT` but not included.

## Behaviour visible in code but not covered by tests

| Behaviour | Observation |
|---|---|
| BSP node loading with `NF_SUBSECTOR` flags | The loader copies `children[]` from WAD as-is. If a WAD has invalid high-bit flags, traversal would misinterpret nodes as subsectors or vice versa — no validation. |
| `R_CheckBBox()` frustum culling | The culling math assumes specific frustum geometry. If view parameters change (e.g., different FOV), the culling might incorrectly skip or include geometry. Not tested. |
| `P_CrossBSPNode()` trace intersection | The trace-vs-seg intersection math in `P_CrossSubsector()` is complex. Edge cases (trace exactly along a seg, trace through a vertex) may have subtle bugs. Not tested. |
| Drawseg/solidseg overflow | `I_Error()` is called on overflow, but the game cannot recover — it exits. No graceful degradation (e.g., dropping far drawsegs). |

## Questions for a human maintainer

1. **Why does the Linux port use `cliprange_t` instead of `solidseg_t`?** Other DOOM ports use the name `solidseg_t` for the same concept. Is this a cleanup by Bernd Kreimeier or an artifact of the porting process?

2. **What is the exact provenance of `NF_SUBSECTOR = 0x8000`?** The 16-bit child encoding limits the theoretical max to 32767 nodes + 32767 subsectors. Was this a deliberate design choice or a consequence of the on-disk format?

3. **Are there any known correctness issues with `R_PointOnSide()` for points exactly on partition lines?** Floating-point / fixed-point edge cases could cause different traversal paths on different platforms.

4. **Does `P_CheckSight()` handle all REJECT table formats correctly?** Some community WADs use different REJECT table conventions (e.g., all-zero for "compute everything"). Is the code robust to this?

5. **Why is the BSP root at `nodes[numnodes - 1]`?** The last node in the array is the root. Is this guaranteed by the node builder, or a convention that could vary between tools?

6. **What happens if `MAXDRAWSEGS`, `MAXVISPLANES`, or `MAXVISSPRITES` overflow in practice?** The hard crash via `I_Error()` suggests these limits were tuned for id Software's maps. Are there known community maps that exceed them?
