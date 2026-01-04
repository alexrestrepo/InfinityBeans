# Appendix K: Save Game File Format

> **Source files**: `game_wad.c`, `tags.h`, `wad.h`
> **Related chapters**: [Chapter 10: File Formats](10_file_formats.md), [Appendix G: Physics File](appendix_g_physics_file.md)

Marathon save games use the standard **WAD file format** with a specific set of tags that capture both map state and runtime game state. This appendix documents the complete save game structure.

---

## Overview

Save games contain a superset of map data — all the original level geometry plus the current state of players, monsters, projectiles, and other dynamic elements. The file type is `'sga∞'` (SAVE_GAME_TYPE).

```
Save Game File Structure:
┌─────────────────────────────────┐
│ WAD Header (128 bytes)          │
│   file_type = 'sga∞'            │
│   wad_count = 1                 │
│   parent_checksum = map file    │
├─────────────────────────────────┤
│ Single WAD containing:          │
│   Level geometry tags           │
│   Runtime state tags            │
│   Physics tags (if modified)    │
├─────────────────────────────────┤
│ WAD Directory (1 entry)         │
└─────────────────────────────────┘
```

**File type constant** (`tags.h:18`):
```c
#define SAVE_GAME_TYPE 'sga∞'
```

---

## Tag Categories

Save games contain two categories of data (`game_wad.c:1408-1450`):

### Level Data Tags (loaded_by_level = TRUE)

These tags represent the static level structure. They're loaded when a level begins and don't change based on player progress:

| Tag | FourCC | Structure | Size | Description |
|-----|--------|-----------|------|-------------|
| `ENDPOINT_DATA_TAG` | `'EPNT'` | `endpoint_data` | 16 bytes | Map vertices |
| `LINE_TAG` | `'LINS'` | `line_data` | 32 bytes | Wall lines |
| `SIDE_TAG` | `'SIDS'` | `side_data` | 64 bytes | Wall textures |
| `POLYGON_TAG` | `'POLY'` | `polygon_data` | 128 bytes | Floor/ceiling polygons |
| `ANNOTATION_TAG` | `'NOTE'` | `map_annotation` | varies | Level comments |
| `OBJECT_TAG` | `'OBJS'` | `map_object` | 16 bytes | Initial object placements |
| `MAP_INFO_TAG` | `'Minf'` | `static_data` | varies | Level metadata |
| `ITEM_PLACEMENT_STRUCTURE_TAG` | `'plac'` | `object_frequency_definition` | varies | Item spawn rules |
| `AMBIENT_SOUND_TAG` | `'ambi'` | `ambient_sound_image_data` | varies | Ambient sounds |
| `RANDOM_SOUND_TAG` | `'bonk'` | `random_sound_image_data` | varies | Random sounds |
| `TERMINAL_DATA_TAG` | `'term'` | byte array | varies | Terminal text |

### Runtime State Tags (loaded_by_level = FALSE)

These tags capture the current game state — positions, health, inventory, etc.:

| Tag | FourCC | Structure | Size | Description |
|-----|--------|-----------|------|-------------|
| `LIGHTSOURCE_TAG` | `'LITE'` | `light_data` | 32 bytes | Current light states |
| `MEDIA_TAG` | `'medi'` | `media_data` | 32 bytes | Liquid levels |
| `MAP_INDEXES_TAG` | `'iidx'` | `short[]` | 2 bytes | Precalculated map data |
| `PLAYER_STRUCTURE_TAG` | `'plyr'` | `player_data` | ~500 bytes | Player state |
| `DYNAMIC_STRUCTURE_TAG` | `'dwol'` | `dynamic_data` | varies | World state counters |
| `OBJECT_STRUCTURE_TAG` | `'mobj'` | `object_data` | 32 bytes | Active objects |
| `AUTOMAP_LINES` | `'alin'` | byte bitfield | varies | Explored lines |
| `AUTOMAP_POLYGONS` | `'apol'` | byte bitfield | varies | Explored polygons |
| `MONSTERS_STRUCTURE_TAG` | `'mOns'` | `monster_data` | 64 bytes | Monster state |
| `EFFECTS_STRUCTURE_TAG` | `'fx  '` | `effect_data` | varies | Visual effects |
| `PROJECTILES_STRUCTURE_TAG` | `'bang'` | `projectile_data` | 32 bytes | Active projectiles |
| `PLATFORM_STRUCTURE_TAG` | `'PLAT'` | `platform_data` | 32 bytes | Platform positions |
| `WEAPON_STATE_TAG` | `'weap'` | byte array | varies | Weapon states |
| `TERMINAL_STATE_TAG` | `'cint'` | byte array | varies | Terminal read state |

### Physics Tags (if modified)

If the game uses modified physics, these are also saved:

| Tag | FourCC | Description |
|-----|--------|-------------|
| `MONSTER_PHYSICS_TAG` | `'MNpx'` | Monster definitions |
| `EFFECTS_PHYSICS_TAG` | `'FXpx'` | Effect definitions |
| `PROJECTILE_PHYSICS_TAG` | `'PRpx'` | Projectile definitions |
| `PHYSICS_PHYSICS_TAG` | `'PXpx'` | Player physics |
| `WEAPONS_PHYSICS_TAG` | `'WPpx'` | Weapon definitions |

---

## Save Game Data Table

The complete data table from `game_wad.c:1416-1450`:

```c
struct save_game_data {
    long tag;
    short unit_size;
    boolean loaded_by_level;
};

struct save_game_data save_data[] = {
    /* Level geometry (loaded_by_level = TRUE) */
    { ENDPOINT_DATA_TAG, sizeof(struct endpoint_data), TRUE },
    { LINE_TAG, sizeof(saved_line), TRUE },
    { SIDE_TAG, sizeof(saved_side), TRUE },
    { POLYGON_TAG, sizeof(saved_poly), TRUE },
    { ANNOTATION_TAG, sizeof(saved_annotation), TRUE },
    { OBJECT_TAG, sizeof(saved_object), TRUE },
    { MAP_INFO_TAG, sizeof(struct static_data), TRUE },
    { ITEM_PLACEMENT_STRUCTURE_TAG,
      MAXIMUM_OBJECT_TYPES*sizeof(struct object_frequency_definition)*2, TRUE },
    { AMBIENT_SOUND_TAG, sizeof(struct ambient_sound_image_data), TRUE },
    { RANDOM_SOUND_TAG, sizeof(struct random_sound_image_data), TRUE },
    { TERMINAL_DATA_TAG, sizeof(byte), TRUE },

    /* Physics (loaded_by_level = TRUE) */
    { MONSTER_PHYSICS_TAG, sizeof(byte), TRUE },
    { EFFECTS_PHYSICS_TAG, sizeof(byte), TRUE },
    { PROJECTILE_PHYSICS_TAG, sizeof(byte), TRUE },
    { PHYSICS_PHYSICS_TAG, sizeof(byte), TRUE },
    { WEAPONS_PHYSICS_TAG, sizeof(byte), TRUE },

    /* Runtime state (loaded_by_level = FALSE) */
    { LIGHTSOURCE_TAG, sizeof(struct light_data), FALSE },
    { MEDIA_TAG, sizeof(struct media_data), FALSE },
    { MAP_INDEXES_TAG, sizeof(short), FALSE },
    { PLAYER_STRUCTURE_TAG, sizeof(struct player_data), FALSE },
    { DYNAMIC_STRUCTURE_TAG, sizeof(struct dynamic_data), FALSE },
    { OBJECT_STRUCTURE_TAG, sizeof(struct object_data), FALSE },
    { AUTOMAP_LINES, sizeof(byte), FALSE },
    { AUTOMAP_POLYGONS, sizeof(byte), FALSE },
    { MONSTERS_STRUCTURE_TAG, sizeof(struct monster_data), FALSE },
    { EFFECTS_STRUCTURE_TAG, sizeof(struct effect_data), FALSE },
    { PROJECTILES_STRUCTURE_TAG, sizeof(struct projectile_data), FALSE },
    { PLATFORM_STRUCTURE_TAG, sizeof(struct platform_data), FALSE },
    { WEAPON_STATE_TAG, sizeof(byte), FALSE },
    { TERMINAL_STATE_TAG, sizeof(byte), FALSE }
};
```

---

## Key Runtime Structures

### Player Data (`player.h`)

The `player_data` structure (~500 bytes) contains:

```c
struct player_data {
    short identifier;              /* unique player ID */
    short flags;                   /* status flags */

    short color, team;             /* appearance */
    char name[MAXIMUM_PLAYER_NAME_LENGTH+1];

    world_point3d location;        /* current position */
    short polygon_index;           /* current polygon */
    angle facing, elevation;       /* view direction */
    short supporting_polygon_index;/* floor polygon */

    world_point3d camera_location; /* actual camera pos */

    short suit_energy, suit_oxygen;/* vitals */
    short monster_index;           /* player's monster slot */
    short object_index;            /* player's object slot */

    /* weapon state */
    short weapon_intensity_decay;
    short weapon_intensity;
    fixed weapon_drawn_portion;
    short current_weapon, desired_weapon;

    /* inventory */
    short items[NUMBER_OF_ITEMS];

    /* more state... */
};
```

### Dynamic World Data (`map.h`)

The `dynamic_data` structure tracks global game state:

```c
struct dynamic_data {
    short tick_count;              /* game ticks elapsed */
    short random_seed;             /* RNG state for determinism */

    struct game_data game_information;  /* game rules */

    short player_count;            /* active players */
    short speaking_player_index;   /* who's talking */

    /* entity counts */
    short polygon_count;
    short side_count;
    short endpoint_count;
    short line_count;
    short lightsource_count;
    short map_index_count;
    short platform_count;

    short object_count;
    short monster_count;
    short projectile_count;
    short effect_count;

    /* current level */
    short current_level_number;

    /* ambient sound state */
    short ambient_sound_image_count;
    short random_sound_image_count;
};
```

### Monster Data (`monsters.h`)

Each monster slot stores:

```c
struct monster_data {
    short type;                    /* monster type index */
    short flags;                   /* state flags */
    short vitality;                /* current health */

    world_point3d location;        /* position */
    short polygon_index;           /* current polygon */
    angle facing;                  /* direction */

    short target_index;            /* who to attack */
    short path_segment_length;     /* pathfinding state */

    /* action state */
    short action, next_action;
    long action_flags;

    /* animation state */
    short sequence, sequence_frame;
    short next_sequence, next_sequence_frame;

    /* vertical position */
    world_distance elevation;
    world_distance desired_height;

    /* external velocity (knockback, etc.) */
    world_vector3d external_velocity;

    /* state timers */
    short ticks_since_last_attack;
    short attack_repetitions;

    /* sound state */
    short sound_index;
    short activation_bias;
};
```

---

## Automap Bitfields

The automap state is stored as packed bitfields:

```c
/* AUTOMAP_LINES: one bit per line */
size = (line_count / 8) + ((line_count % 8) ? 1 : 0);

/* AUTOMAP_POLYGONS: one bit per polygon */
size = (polygon_count / 8) + ((polygon_count % 8) ? 1 : 0);

/* bit set = player has explored this line/polygon */
boolean is_explored(byte *automap_data, short index) {
    return (automap_data[index >> 3] & (1 << (index & 7))) != 0;
}
```

---

## Save Process

The save process (`game_wad.c:1004-1082`):

```c
boolean save_game_file(FileDesc *file) {
    struct wad_header header;
    struct wad_data *wad;
    long wad_length;

    /* save current random seed for deterministic restore */
    dynamic_world->random_seed = get_random_seed();

    /* create header with parent checksum linking to original map */
    fill_default_wad_header(file, CURRENT_WADFILE_VERSION,
        EDITOR_MAP_VERSION, 1, 0, &header);

    /* create file with save game type */
    create_wadfile(file, SAVE_GAME_TYPE);

    /* build wad from all game arrays */
    wad = build_save_game_wad(&header, &wad_length);

    /* write header, wad, and directory */
    write_wad_header(file_ref, &header);
    write_wad(file_ref, &header, wad, offset);

    /* set parent checksum so we can find the original map */
    header.parent_checksum = read_wad_file_checksum(&current_map_file);
    write_wad_header(file_ref, &header);  /* rewrite with checksum */
    write_directorys(file_ref, &header, &entry);

    return success;
}
```

The `build_save_game_wad` function (`game_wad.c:1593-1621`) iterates through all tags:

```c
static struct wad_data *build_save_game_wad(
    struct wad_header *header,
    long *length)
{
    struct wad_data *wad;
    short loop;
    byte *array;
    long size;

    wad = create_empty_wad();
    if (wad) {
        recalculate_map_counts();

        for (loop = 0; loop < NUMBER_OF_SAVE_ARRAYS; ++loop) {
            /* get pointer to data and its size */
            array = tag_to_global_array_and_size(
                save_data[loop].tag, &size);

            /* add to wad if size > 0 */
            if (size) {
                wad = append_data_to_wad(wad,
                    save_data[loop].tag, array, size, 0);
            }
        }

        *length = calculate_wad_length(header, wad);
    }

    return wad;
}
```

---

## Load Process

Loading a save game (`game_wad.c:902-947`):

```c
boolean load_game_from_file(FileDesc *file) {
    boolean success = FALSE;

    /* saved games are single-player */
    game_is_networked = FALSE;

    /* use save file as map source */
    set_map_file(file);

    /* load level (index NONE = saved game) */
    success = load_level_from_map(NONE);

    if (success) {
        /* find original map using parent checksum */
        unsigned long parent_checksum =
            read_wad_file_parent_checksum(file);

        if (!use_map_file(parent_checksum)) {
            /* warn user - can't switch levels without original map */
            alert_user(infoError, strERRORS, cantFindMap, 0);
            set_to_default_map();
        }

        /* restore deterministic random state */
        set_random_seed(dynamic_world->random_seed);

        /* load graphics and start game */
        entering_map();
    }

    return success;
}
```

The `complete_restoring_level` function (`game_wad.c:1624-1651`) loads runtime tags:

```c
static void complete_restoring_level(struct wad_data *wad) {
    short loop;
    void *array;
    byte *data;
    long size, data_length;

    for (loop = 0; loop < NUMBER_OF_SAVE_ARRAYS; ++loop) {
        /* skip tags already loaded by level processing */
        if (!save_data[loop].loaded_by_level) {
            /* get destination array */
            array = tag_to_global_array_and_size(
                save_data[loop].tag, &size);

            /* extract data from wad */
            data = extract_type_from_wad(wad,
                save_data[loop].tag, &data_length);

            /* copy directly to global array */
            memcpy(array, data, data_length);
        }
    }

    /* reset input queues for fresh start */
    reset_player_queues();
}
```

---

## Parent Checksum

The `parent_checksum` field in the WAD header links the save to its original map file. This allows:

1. Finding the correct map when loading
2. Enabling level transitions (player finishes current level)
3. Verifying save game validity

```c
/* on save */
header.parent_checksum = read_wad_file_checksum(&current_map_file);

/* on load */
unsigned long parent_checksum = read_wad_file_parent_checksum(file);
if (!use_map_file(parent_checksum)) {
    /* can't find original map */
}
```

---

## Parsing Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* byte swapping for big-endian data */
short swap16(short val) {
    return ((val & 0xFF) << 8) | ((val >> 8) & 0xFF);
}

long swap32(long val) {
    return ((val & 0xFF) << 24) | ((val & 0xFF00) << 8) |
           ((val >> 8) & 0xFF00) | ((val >> 24) & 0xFF);
}

boolean parse_save_game(char *filename) {
    FILE *f;
    struct wad_header header;
    struct wad_data *wad;
    long data_length;
    void *data;

    f = fopen(filename, "rb");
    if (!f) return FALSE;

    /* read and byte-swap header */
    fread(&header, sizeof(struct wad_header), 1, f);
    header.version = swap16(header.version);
    header.wad_count = swap16(header.wad_count);
    header.directory_offset = swap32(header.directory_offset);
    header.parent_checksum = swap32(header.parent_checksum);

    printf("Save game info:\n");
    printf("  Version: %d\n", header.version);
    printf("  Parent checksum: %08lX\n", header.parent_checksum);

    /* read the single wad (save games always have 1 wad) */
    wad = read_indexed_wad_from_file_handle(f, &header, 0);
    if (!wad) {
        fclose(f);
        return FALSE;
    }

    /* extract player data */
    data = extract_type_from_wad(wad, PLAYER_STRUCTURE_TAG, &data_length);
    if (data) {
        short player_count = data_length / sizeof(struct player_data);
        struct player_data *players = (struct player_data *)data;

        printf("  Players: %d\n", player_count);
        for (short i = 0; i < player_count; i++) {
            printf("    Player %d: %s (health: %d)\n",
                i, players[i].name,
                swap16(players[i].suit_energy));
        }
    }

    /* extract dynamic world data */
    data = extract_type_from_wad(wad, DYNAMIC_STRUCTURE_TAG, &data_length);
    if (data) {
        struct dynamic_data *dw = (struct dynamic_data *)data;
        printf("  Tick count: %d\n", swap16(dw->tick_count));
        printf("  Level: %d\n", swap16(dw->current_level_number));
    }

    /* extract monster count */
    data = extract_type_from_wad(wad, MONSTERS_STRUCTURE_TAG, &data_length);
    if (data) {
        short monster_count = data_length / sizeof(struct monster_data);
        printf("  Active monsters: %d\n", monster_count);
    }

    free_wad(wad);
    fclose(f);
    return TRUE;
}
```

---

## See Also

- [Chapter 10: File Formats](10_file_formats.md) — WAD file structure
- [Appendix G: Physics File Format](appendix_g_physics_file.md) — Physics tag details
- [Appendix H: Film Format](appendix_h_film_format.md) — Recording format (similar WAD usage)
- [Appendix I: Cheat Sheet](appendix_i_cheatsheet.md) — Structure sizes
