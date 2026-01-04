# Appendix J: Modding Cookbook

> Step-by-step recipes for common Marathon modifications.
> Each recipe includes source file references and tested approaches.

---

## Recipe 1: Modify Player Speed

**Goal**: Make the player move faster or slower.

**Files involved**: `physics_models.h`, physics file (`.phy∞`)

**Method A: Source code modification**

Edit `physics_models.h:42-43` (walking model) or `:60-61` (running model):

```c
// original walking speed
FIXED_ONE/14,  // maximum_forward_velocity  (~4681)
FIXED_ONE/17,  // maximum_backward_velocity (~3855)

// doubled speed
FIXED_ONE/7,   // maximum_forward_velocity  (~9362)
FIXED_ONE/8,   // maximum_backward_velocity (~8192)
```

**Method B: External physics file**

1. Create a physics file with modified `'PXpx'` tag
2. Export the `physics_constants` structure with new values
3. Place in game folder; game warns "external physics model"

**Values to change**:
| Field | Effect |
|-------|--------|
| `maximum_forward_velocity` | Top forward speed |
| `maximum_backward_velocity` | Top backward speed |
| `maximum_perpendicular_velocity` | Top strafe speed |
| `acceleration` | How fast you reach top speed |
| `deceleration` | How fast you stop |

**Conversion**: `FIXED_ONE = 65536`, so `FIXED_ONE/14 = 4681`

---

## Recipe 2: Modify Weapon Damage

**Goal**: Make weapons deal more or less damage.

**Files involved**: `weapon_definitions.h`, `projectile_definitions.h`

**Understanding the chain**:
```
Weapon → fires → Projectile → deals → Damage
```

Weapons reference projectiles; projectiles define damage.

**Step 1**: Find the weapon in `weapon_definitions.h`

```c
// example: pistol (weapon index 1)
{
    // ... other fields ...
    _projectile_pistol_bullet,  // projectile type
    // ...
}
```

**Step 2**: Find the projectile in `projectile_definitions.h`

```c
// _projectile_pistol_bullet
{
    // ... other fields ...
    {_damage_projectile, 0, 20, 5},  // damage: type, flags, base, random
    // ...
}
```

**Step 3**: Modify the damage tuple

```c
// damage structure: {type, flags, base, random}
// actual damage = base + random_value(0, random)

// original pistol: 20 + 0-5 = 20-25 damage
{_damage_projectile, 0, 20, 5}

// buffed pistol: 40 + 0-10 = 40-50 damage
{_damage_projectile, 0, 40, 10}
```

**Damage types** (from `map.h`):
| Type | ID | Notes |
|------|-----|-------|
| `_damage_projectile` | 2 | Standard bullets |
| `_damage_explosion` | 0 | Splash damage |
| `_damage_flame` | 4 | Fire damage |
| `_damage_fusion_bolt` | 9 | Fusion pistol |

---

## Recipe 3: Modify Monster Health

**Goal**: Make monsters tougher or weaker.

**Files involved**: `monster_definitions.h`

**Find the monster** by index (see `monsters.h` for enum):

```c
// example: Pfhor Fighter (index varies by type)
{
    // ... many fields ...
    80,     // vitality (health points)
    // ...
}
```

**Modify vitality**:

```c
// weaker fighter (dies faster)
40,     // vitality

// tougher fighter (bullet sponge)
200,    // vitality
```

**Monster indices** (partial list):
| Index | Monster |
|-------|---------|
| 0-3 | Pfhor Fighters (minor to major) |
| 4-6 | Pfhor Troopers |
| 14-16 | Hunters |
| 20-22 | Enforcers |
| 33-35 | BOBs (civilians) |

---

## Recipe 4: Modify Gravity

**Goal**: Low gravity moon level or high gravity challenge.

**Files involved**: `physics_models.h`

**Find gravity constant** (`physics_models.h:44` or `:64`):

```c
FIXED_ONE/400,  // gravitational_acceleration (default: ~164)
```

**Modifications**:

```c
// low gravity (floaty jumps)
FIXED_ONE/800,  // half gravity (~82)

// high gravity (heavy, short jumps)
FIXED_ONE/200,  // double gravity (~328)

// moon gravity (~1/6 Earth)
FIXED_ONE/2400, // (~27)
```

**Also adjust**:
- `terminal_velocity` — max falling speed
- `climbing_acceleration` — affects jump height feel

---

## Recipe 5: Parse a Map File

**Goal**: Extract level data programmatically.

**Files involved**: `wad.h`, `wad.c`, `tags.h`

**Implementation**:

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

/* wad header structure (128 bytes) */
struct wad_header {
    short version;
    short data_version;
    char file_name[64];
    unsigned long checksum;
    long directory_offset;
    short wad_count;
    short application_specific_directory_data_size;
    short entry_header_size;
    short directory_entry_base_size;
    unsigned long parent_checksum;
    short unused[20];
};

/* directory entry (10 bytes + app-specific) */
struct directory_entry {
    long offset_to_start;
    long length;
    short index;
};

boolean parse_marathon_map(char *filename) {
    FILE *f;
    struct wad_header header;
    struct directory_entry *entries;
    short i;

    f = fopen(filename, "rb");
    if (!f) return FALSE;

    /* 1. read wad header */
    fread(&header, sizeof(struct wad_header), 1, f);
    header.version = swap16(header.version);
    header.wad_count = swap16(header.wad_count);
    header.directory_offset = swap32(header.directory_offset);

    /* 2. seek to directory */
    fseek(f, header.directory_offset, SEEK_SET);

    /* 3. read directory entries */
    entries = malloc(header.wad_count * sizeof(struct directory_entry));
    for (i = 0; i < header.wad_count; i++) {
        fread(&entries[i], 10, 1, f);  /* base size */
        entries[i].offset_to_start = swap32(entries[i].offset_to_start);
        entries[i].length = swap32(entries[i].length);
        entries[i].index = swap16(entries[i].index);
    }

    /* 4. for each level, read tagged chunks */
    for (i = 0; i < header.wad_count; i++) {
        long position = entries[i].offset_to_start;
        long end_position = position + entries[i].length;

        fseek(f, position, SEEK_SET);

        while (position < end_position) {
            long tag, next_offset, length;
            void *data;

            fread(&tag, 4, 1, f);
            fread(&next_offset, 4, 1, f);
            fread(&length, 4, 1, f);
            next_offset = swap32(next_offset);
            length = swap32(length);

            data = malloc(length);
            fread(data, length, 1, f);

            /* process by tag type */
            if (tag == 'PNTS') {
                /* parse points */
            } else if (tag == 'LINS') {
                /* parse lines */
            } else if (tag == 'POLY') {
                /* parse polygons */
            }
            /* ... etc */

            free(data);
            position = next_offset;
            fseek(f, position, SEEK_SET);
        }
    }

    free(entries);
    fclose(f);
    return TRUE;
}
```

**Key structures** (sizes for parsing):
| Tag | Structure | Size |
|-----|-----------|------|
| `'PNTS'` | `world_point2d` | 4 bytes |
| `'EPNT'` | `endpoint_data` | 16 bytes |
| `'LINS'` | `line_data` | 32 bytes |
| `'SIDS'` | `side_data` | 64 bytes |
| `'POLY'` | `polygon_data` | 128 bytes |
| `'LITE'` | `light_data` | 32 bytes |
| `'OBJS'` | `object_data` | 16 bytes |

**Remember**: All data is big-endian!

---

## Recipe 6: Parse a Film File

**Goal**: Extract gameplay recording data.

**Files involved**: `vbl_definitions.h`, `vbl.c`

**Implementation**:

```c
#include <stdio.h>
#include <stdlib.h>

#define MAXIMUM_NUMBER_OF_PLAYERS 8
#define MAXIMUM_PLAYER_START_NAME_LENGTH 32
#define END_OF_RECORDING_INDICATOR 257

struct player_start_data {
    short team;
    short identifier;
    short color;
    char name[MAXIMUM_PLAYER_START_NAME_LENGTH + 1];
};

struct game_data {
    long game_time_remaining;
    short game_type;
    short game_options;
    short kill_limit;
    short initial_random_seed;
    short difficulty_level;
    short parameters[2];
};

struct recording_header {
    long length;
    short num_players;
    short level_number;
    unsigned long map_checksum;
    short version;
    struct player_start_data starts[MAXIMUM_NUMBER_OF_PLAYERS];
    struct game_data game_information;
};

void decode_action_flags(long flags) {
    if (flags & (1 << 15)) printf("forward ");
    if (flags & (1 << 16)) printf("backward ");
    if (flags & (1 << 22)) printf("strafe_left ");
    if (flags & (1 << 23)) printf("strafe_right ");
    if (flags & (1 << 24)) printf("fire_primary ");
    if (flags & (1 << 25)) printf("fire_secondary ");
    if (flags & (1 << 26)) printf("action ");
    if (flags & (1 << 1))  printf("turn_left ");
    if (flags & (1 << 2))  printf("turn_right ");
    printf("\n");
}

boolean parse_marathon_film(char *filename) {
    FILE *f;
    struct recording_header header;
    long tick;
    short player;

    f = fopen(filename, "rb");
    if (!f) return FALSE;

    /* 1. read header */
    fread(&header, sizeof(struct recording_header), 1, f);
    header.length = swap32(header.length);
    header.num_players = swap16(header.num_players);
    header.level_number = swap16(header.level_number);
    header.map_checksum = swap32(header.map_checksum);
    header.version = swap16(header.version);

    printf("Players: %d\n", header.num_players);
    printf("Level: %d\n", header.level_number);
    printf("Map checksum: %08lX\n", header.map_checksum);

    /* 2. read action flags (RLE encoded) */
    tick = 0;
    while (1) {
        for (player = 0; player < header.num_players; player++) {
            short count;
            long flags;
            short i;

            fread(&count, sizeof(short), 1, f);
            count = swap16(count);

            if (count == END_OF_RECORDING_INDICATOR) {
                printf("End of recording at tick %ld\n", tick);
                fclose(f);
                return TRUE;
            }

            fread(&flags, sizeof(long), 1, f);
            flags = swap32(flags);

            for (i = 0; i < count; i++) {
                printf("Tick %ld, Player %d: %08lX ", tick, player, flags);
                decode_action_flags(flags);
                tick++;
            }
        }
    }

    fclose(f);
    return TRUE;
}
```

---

## Recipe 7: Create a Physics Patch

**Goal**: Distribute physics changes without replacing entire file.

**Concept**: Store only the bytes that changed.

**Structure**:
```
Patch WAD:
  header.parent_checksum = original file's checksum
  For each changed tag:
    tag + offset_of_first_change + changed_bytes_only
```

**Using physics_patches.c**:

```bash
# compile the tool
cc -o physics_patches physics_patches.c wad.c ...

# create patch
./physics_patches original.phy∞ modified.phy∞ patch.pat2
```

**Manual implementation**:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define NUMBER_OF_PHYSICS_TAGS 5

static long physics_tags[NUMBER_OF_PHYSICS_TAGS] = {
    'MNpx', 'FXpx', 'PRpx', 'PXpx', 'WPpx'
};

boolean create_physics_patch(char *original_file, char *modified_file, char *patch_file) {
    struct wad_data *orig_wad, *mod_wad, *patch_wad;
    unsigned long parent_checksum;
    short i;
    boolean had_changes = FALSE;

    /* load both physics files */
    orig_wad = load_physics_wad(original_file, &parent_checksum);
    mod_wad = load_physics_wad(modified_file, NULL);
    if (!orig_wad || !mod_wad) return FALSE;

    patch_wad = create_empty_wad();

    /* compare each tag */
    for (i = 0; i < NUMBER_OF_PHYSICS_TAGS; i++) {
        long tag = physics_tags[i];
        long orig_length, mod_length;
        byte *orig_data, *mod_data;
        long first_diff, last_diff, offset;

        orig_data = extract_type_from_wad(orig_wad, tag, &orig_length);
        mod_data = extract_type_from_wad(mod_wad, tag, &mod_length);

        if (!orig_data || !mod_data || orig_length != mod_length) {
            continue;
        }

        /* find first and last differing bytes */
        first_diff = NONE;
        last_diff = 0;
        for (offset = 0; offset < orig_length; offset++) {
            if (orig_data[offset] != mod_data[offset]) {
                if (first_diff == NONE) {
                    first_diff = offset;
                }
                last_diff = offset;
                had_changes = TRUE;
            }
        }

        /* if differences found, add to patch */
        if (first_diff != NONE) {
            long patch_length = last_diff - first_diff + 1;
            patch_wad = append_data_to_wad(patch_wad, tag,
                &mod_data[first_diff], patch_length, first_diff);
        }
    }

    /* write patch file if there were changes */
    if (had_changes) {
        write_patch_wad(patch_file, patch_wad, parent_checksum);
    }

    free_wad(orig_wad);
    free_wad(mod_wad);
    free_wad(patch_wad);

    return had_changes;
}
```

**Key point**: The `append_data_to_wad` function stores the offset along with the data, so the game knows where to apply the patch bytes.

---

## Recipe 8: Add Custom Sounds

**Goal**: Replace or add sound effects.

**Files involved**: Sound file format, `sound.c`

**Sound file structure**:
- Header with sound count
- For each sound: header + 8-bit or 16-bit PCM data
- Sounds indexed by `sound_definitions.h` enums

**Steps**:

1. **Identify sound index** from `sound_definitions.h`:
   ```c
   enum {
       _snd_startup,           // 0
       _snd_teleport_in,       // 1
       _snd_pistol_fire,       // 10
       // ... etc
   };
   ```

2. **Prepare audio**:
   - Convert to 8-bit unsigned or 16-bit signed PCM
   - Sample rate: 22050 Hz typical
   - Mono

3. **Replace in sound file**:
   - Parse existing sound file
   - Replace data at target index
   - Recalculate offsets
   - Write modified file

**Tool recommendation**: Use Anvil (Bungie's editor) or community tools like ShapeFusion for sound editing.

---

## Recipe 9: Understand Memory Layout

**Goal**: Debug or extend data structures.

**Key arrays** (from `map.c` globals):

```c
// world geometry (allocated per level)
struct endpoint_data *endpoints;        // MAXIMUM_ENDPOINTS_PER_MAP
struct line_data *lines;                // MAXIMUM_LINES_PER_MAP
struct side_data *sides;                // MAXIMUM_SIDES_PER_MAP
struct polygon_data *polygons;          // MAXIMUM_POLYGONS_PER_MAP

// dynamic objects (object slots)
struct object_data *objects;            // MAXIMUM_OBJECTS_PER_MAP
struct monster_data *monsters;          // MAXIMUM_MONSTERS_PER_MAP
struct projectile_data *projectiles;    // MAXIMUM_PROJECTILES_PER_MAP
struct effect_data *effects;            // MAXIMUM_EFFECTS_PER_MAP

// players
struct player_data *players;            // MAXIMUM_NUMBER_OF_PLAYERS
```

**Object slot system**:

```c
// objects use linked lists for efficiency
struct object_data {
    short object_type;          // type identifier
    short object_index;         // index into type-specific array
    // ... position, polygon, etc ...
    short next_object;          // next in polygon's object list
};

// finding objects in a polygon
short object_index = polygon->first_object;
while (object_index != NONE) {
    struct object_data *obj = objects + object_index;
    // process object
    object_index = obj->next_object;
}
```

---

## Recipe 10: Debug Network Desync

**Goal**: Find why multiplayer games desync.

**Common causes**:

1. **Floating-point usage** — Must use fixed-point only
2. **Uninitialized memory** — Different values on different machines
3. **Platform differences** — Byte order, struct padding
4. **Random number misuse** — Using system RNG instead of game RNG

**Debugging approach**:

```c
// add checksum validation (from network.c concept)
unsigned long calculate_game_state_checksum(void) {
    unsigned long checksum = 0;

    // hash player positions
    for (int i = 0; i < dynamic_world->player_count; i++) {
        struct player_data *p = players + i;
        checksum ^= p->location.x;
        checksum ^= p->location.y;
        checksum ^= p->location.z;
        checksum = (checksum << 1) | (checksum >> 31);
    }

    // hash monster positions
    for (int i = 0; i < MAXIMUM_MONSTERS_PER_MAP; i++) {
        if (SLOT_IS_USED(monsters + i)) {
            // add to checksum
        }
    }

    return checksum;
}

// compare every N ticks
if ((tick_count % 10) == 0) {
    send_checksum_to_all_players(calculate_game_state_checksum());
}
```

**Prevention checklist**:
- [ ] All math uses `fixed` type
- [ ] All random calls use `global_random()`
- [ ] No `float` or `double` anywhere in simulation
- [ ] Structures are packed consistently
- [ ] Byte order handled for all file I/O

---

## See Also

- [Appendix G: Physics File Format](appendix_g_physics_file.md) — Complete physics file spec
- [Appendix H: Film Format](appendix_h_film_format.md) — Film file parsing details
- [Appendix I: Cheat Sheet](appendix_i_cheatsheet.md) — Quick constant reference
- [Chapter 10: File Formats](10_file_formats.md) — WAD format details
- [Cross-References](cross_references.md) — Find related topics
