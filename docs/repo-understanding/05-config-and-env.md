# BSP-relevant configuration and environment

There is no standalone BSP configuration file. The subsystem is mostly controlled by the selected WAD/map data and compile-time assumptions.

## WAD and map selection

| Input | BSP relevance |
|---|---|
| `DOOMWADDIR` environment variable | Directory searched for known IWAD names such as `doom.wad`, `doomu.wad`, and `doom2.wad`; the selected IWAD supplies map BSP lumps. Defaults to `.`. |
| `-file <wad...>` | Adds PWADs after the IWAD. Later lumps override earlier same-named lumps in the WAD directory. |
| `-warp` | Chooses the initial episode/map or commercial map, determining the marker used by `P_SetupLevel`. |
| `-episode` | Selects episode and starts map 1 for non-commercial formats. |
| `-wart` | Development-map shortcut that adds a reloadable WAD and rewrites itself to `-warp`. |
| Game mode / detected IWAD | Chooses `E#M#` versus `MAP##` marker naming. |

`-devparm` and `-debugfile` exist, but no BSP-specific debug output or visualization is connected to them in the inspected code.

## Data-driven behavior

The actual partition lines, node hierarchy, child boxes, seg splits, subsectors, sector-pair rejection, and block grid all come from map lumps. This executable consumes them; it does not expose runtime knobs for rebuilding or changing the BSP.

## Hardcoded format and numeric assumptions

| Assumption | Location / effect |
|---|---|
| Classic ordered Doom map lumps | `doomdata.h` defines `ML_*`; `P_SetupLevel` loads by offsets after a map marker. |
| Packed map fields are mostly 16-bit values | `map*` structures in `doomdata.h`; loader counts are lump length divided by record size. |
| Little-endian conversion is required | Loader fields pass through `SHORT`; WAD headers/directory fields use `LONG`. |
| Runtime map coordinates use signed 16.16 fixed point | `FRACBITS` is 16 in `m_fixed.h`; setup shifts map coordinates/heights/offsets. |
| Root node is last | Rendering, point lookup, and sight start at `numnodes - 1`. |
| High child bit marks a subsector | `NF_SUBSECTOR` is `0x8000`; remaining bits identify the leaf. |
| Single-subsector map special case | `R_PointInSubsector` returns subsector zero when there are no nodes; render/sight code also recognize special leaf handling. |
| Subsector's first seg identifies its sector | `P_GroupLines` assigns `ss->sector = first_seg->sidedef->sector`. |
| BLOCKMAP cells are 128 map units | `MAPBLOCKUNITS` and related shifts in `p_local.h`. |
| Fixed renderer capacities | `MAXDRAWSEGS` is 256, `MAXVISPLANES` is 128, and openings are `SCREENWIDTH * 64`. Overflow handling varies. |
| Fixed base screen dimensions | Screen-column clipping and visplane arrays are tied to `SCREENWIDTH` 320 / `SCREENHEIGHT` 200. |
| Optional range checks are compiled in by default | `RANGECHECK` is defined in `doomdef.h`; it guards some subsector and renderer validations. |

## Level-data lifetime and reload behavior

- Runtime BSP arrays use the zone allocator with `PU_LEVEL` and are discarded on the next level setup.
- `REJECT` and `BLOCKMAP` are cached with level lifetime.
- A development WAD prefixed internally with `~` is reloadable; `P_SetupLevel` calls `W_Reload` before resolving/loading map data.
- `precache` controls graphics preloading after map setup, not BSP loading or traversal.

## Test fixture configuration

No test runner, fixture configuration, WAD fixture, or BSP-specific test flag is present in the repository. Exercising BSP behavior requires an external compatible IWAD/PWAD.
