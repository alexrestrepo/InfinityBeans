# Appendix E: Marathon 2 vs Marathon Infinity

## Source Code Differences

---

## E.1 Overview

The source code in this repository is **Marathon Infinity**, which is a superset of Marathon 2. All code resides in the `marathon2/` directory. This appendix documents the differences between the two releases.

> **From `marathon_src/README.md`:**
>
> *"A limited commit history was reconstructed from the two releases, integrating earlier versions and obsolete files present in the Infinity source archive."*
>
> *"When Marathon 2's source code was released in 2000, some components of the game engine were still commercially relevant, and were excluded from the public release."*

---

## E.2 Monster Additions

Marathon Infinity added **4 vacuum civilian variants** for space environments:

```c
// From monsters.h:83-87
enum /* monster types (partial - showing Infinity additions) */ {
    // ... Marathon 2 monsters (0-42) ...
    _monster_tiny_yeti,           // 42 - Last M2 monster
    _vacuum_civilian_crew,        // 43 - INFINITY ONLY
    _vacuum_civilian_science,     // 44 - INFINITY ONLY
    _vacuum_civilian_security,    // 45 - INFINITY ONLY
    _vacuum_civilian_assimilated, // 46 - INFINITY ONLY
    NUMBER_OF_MONSTER_TYPES       // 47
};
```

| Version | Monster Count | Last Monster Type |
|---------|---------------|-------------------|
| Marathon 2 | 43 | `_monster_tiny_yeti` |
| Marathon Infinity | 47 | `_vacuum_civilian_assimilated` |

---

## E.3 Collection Additions

Infinity added a new shape collection for vacuum civilians:

```c
// From shape_descriptors.h:42
_collection_vacuum_civilian, // 13
```

And corresponding effects:

```c
// From effects.h:81-82
_effect_vacuum_civilian_blood_splash,
_effect_assimilated_vacuum_civilian_blood_splash,
```

---

## E.4 Map/WAD Version Numbers

```c
// From editor.h:10-12
#define MARATHON_ONE_DATA_VERSION 0
#define MARATHON_TWO_DATA_VERSION 1
#define MARATHON_INFINITY_DATA_VERSION 2

// From wad.h:17-22
// The Infinity demo was version 3.
// Infinity release will be version 4.
#define CURRENT_WADFILE_VERSION 2
#ifdef DEBUG
    #define INFINITY_WADFILE_VERSION (CURRENT_WADFILE_VERSION+1)  // 3
#else
    #define INFINITY_WADFILE_VERSION (CURRENT_WADFILE_VERSION+2)  // 4
#endif
```

The code handles all three versions:

```c
// From game_wad.c:1135
assert(version==MARATHON_INFINITY_DATA_VERSION ||
       version==MARATHON_TWO_DATA_VERSION ||
       version==MARATHON_ONE_DATA_VERSION);
```

---

## E.5 Network Protocol Versions

```c
// From network_dialogs.c:58-60
// changed to "10" for marathon infinity
// changed to "11" for Infinity Demo with InputSprocket support
// changed to "12" for Infinity Full with InputSprocket (Trilogy)
```

This prevents incompatible clients from connecting to each other.

---

## E.6 Vacuum Environment Support

Infinity enhanced vacuum environment handling:

```c
// From map.h:673
_environment_vacuum = 0x0001, // prevents certain weapons from working, player uses oxygen

// From player.c:437
if ((static_world->environment_flags & _environment_vacuum) ||
    (player->variables.flags & _HEAD_BELOW_MEDIA_BIT))
{
    handle_player_in_vacuum(player_index, action_flags);
}
```

Items that don't work in vacuum are flagged:

```c
// From item_definitions.h:34-53 (partial)
{_weapon, 8, 8, BUILD_DESCRIPTOR(_collection_items, 2), 1, _environment_vacuum},
// ... weapons and ammo with vacuum restrictions
```

---

## E.7 Files Edited for Marathon 2 Release

When Marathon 2's source was released in 2000, these files were edited to remove commercially sensitive code (serial number generation, cseries.lib integration):

- `game_dialogs.c`
- `game_wad.c`
- `interface.c` / `interface.h`
- `makefile`
- `player.c`
- `preferences.c`
- `shell.c`

The 2011 Infinity release was more complete.

---

## E.8 Practical Implications for Porting

| Consideration | Recommendation |
|---------------|----------------|
| **Which source to use?** | Use Infinity (this repo)—it's more complete |
| **Loading M2 maps in Infinity port?** | Works automatically via version checks in `game_wad.c` |
| **Monster count** | Use `NUMBER_OF_MONSTER_TYPES` (47), not hardcoded 43 |
| **Vacuum civilians** | Optional—only appear in Infinity scenarios |
| **Network compatibility** | M2 and Infinity clients cannot play together |

---

## E.9 Feature Comparison Summary

| Feature | Marathon 2 | Marathon Infinity |
|---------|-----------|-------------------|
| Monster types | 43 | 47 (+4 vacuum civilians) |
| Shape collections | 12 | 13 (+vacuum_civilian) |
| Map data version | 1 | 2 |
| WAD file version | 2 | 3 (demo) / 4 (release) |
| Network version | Unknown | 10-12 |
| Vacuum civilians | No | Yes |
| InputSprocket | No | Yes (v11+) |
| License | GPL 2 | GPL 3 |

---

*Return to: [Appendix D: Fixed-Point](appendix_d_fixedpoint.md) | [Table of Contents](README.md)*
