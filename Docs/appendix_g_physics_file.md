# Appendix G: Physics File Format

> **Source files**: `extensions.h`, `import_definitions.c`, `physics_patches.c`, `physics_models.h`
> **Related chapters**: [Chapter 6: Physics](06_physics.md), [Chapter 8: Entities](08_entities.md)

Marathon allows external modification of game physics through **physics files** (`.phy∞` or `.phyA`). This appendix documents the complete file format for creating or parsing physics modifications.

---

## Overview

Physics files use the standard **Marathon WAD format** (see [Chapter 10](10_file_formats.md)) with specific tagged chunks for each physics subsystem. Unlike map WADs which contain level geometry, physics WADs contain arrays of game constants.

```
Physics File Structure:
┌─────────────────────────────────┐
│ WAD Header (128 bytes)          │
│   data_version = 0 or 1         │
├─────────────────────────────────┤
│ Tagged Chunk: 'MNpx'            │  ← Monster definitions
│ Tagged Chunk: 'FXpx'            │  ← Effect definitions
│ Tagged Chunk: 'PRpx'            │  ← Projectile definitions
│ Tagged Chunk: 'PXpx'            │  ← Player physics constants
│ Tagged Chunk: 'WPpx'            │  ← Weapon definitions
├─────────────────────────────────┤
│ WAD Directory                   │
└─────────────────────────────────┘
```

**File type constants** (`tags.h:20`):
```c
#define PHYSICS_FILE_TYPE 'phy∞'  // full physics file
#define PATCH_FILE_TYPE   'pat2'  // delta patch file
```

**Data version** (`extensions.h:8-9`):
```c
#define BUNGIE_PHYSICS_DATA_VERSION 0  // original Bungie physics
#define PHYSICS_DATA_VERSION        1  // user-modified physics
```

---

## Physics Tags

Each physics subsystem has its own tag (`tags.h:60-64`):

| Tag | FourCC | Contents | Count | Size per Entry |
|-----|--------|----------|-------|----------------|
| `MONSTER_PHYSICS_TAG` | `'MNpx'` | Monster definitions | 47 | ~156 bytes |
| `EFFECTS_PHYSICS_TAG` | `'FXpx'` | Visual effect definitions | 60 | ~12 bytes |
| `PROJECTILE_PHYSICS_TAG` | `'PRpx'` | Projectile definitions | 43 | ~48 bytes |
| `PHYSICS_PHYSICS_TAG` | `'PXpx'` | Player physics models | 2 | 88 bytes |
| `WEAPONS_PHYSICS_TAG` | `'WPpx'` | Weapon definitions | 10 | ~160 bytes |

The definition table (`extensions.h:20-28`):
```c
static struct definition_data definitions[] = {
    {MONSTER_PHYSICS_TAG, monster_definitions, NUMBER_OF_MONSTER_TYPES, sizeof(struct monster_definition)},
    {EFFECTS_PHYSICS_TAG, effect_definitions, NUMBER_OF_EFFECT_TYPES, sizeof(struct effect_definition)},
    {PROJECTILE_PHYSICS_TAG, projectile_definitions, NUMBER_OF_PROJECTILE_TYPES, sizeof(struct projectile_definition)},
    {PHYSICS_PHYSICS_TAG, physics_models, NUMBER_OF_PHYSICS_MODELS, sizeof(struct physics_constants)},
    {WEAPONS_PHYSICS_TAG, weapon_definitions, MAXIMUM_NUMBER_OF_WEAPONS, sizeof(struct weapon_definition)}
};
```

---

## Player Physics Constants

The `physics_constants` structure controls player movement (`physics_models.h:17-34`):

```c
struct physics_constants {
    // linear movement (fixed-point, FIXED_ONE = 65536)
    fixed maximum_forward_velocity;      // max speed moving forward
    fixed maximum_backward_velocity;     // max speed moving backward
    fixed maximum_perpendicular_velocity;// max strafe speed
    fixed acceleration;                  // forward/backward accel
    fixed deceleration;                  // friction when stopping
    fixed airborne_deceleration;         // air control friction
    fixed gravitational_acceleration;    // falling acceleration
    fixed climbing_acceleration;         // stair climb rate
    fixed terminal_velocity;             // max falling speed
    fixed external_deceleration;         // decel from external forces

    // angular movement (fixed-point angles)
    fixed angular_acceleration;          // turn acceleration
    fixed angular_deceleration;          // turn deceleration
    fixed maximum_angular_velocity;      // max turn speed
    fixed angular_recentering_velocity;  // auto-center speed
    fixed fast_angular_velocity;         // fast look speed
    fixed fast_angular_maximum;          // fast look max
    fixed maximum_elevation;             // max look up/down angle
    fixed external_angular_deceleration; // external turn friction

    // step animation
    fixed step_delta;                    // distance per step
    fixed step_amplitude;                // vertical bob amount

    // collision geometry
    fixed radius;                        // player collision radius
    fixed height;                        // standing height
    fixed dead_height;                   // dead/crouched height
    fixed camera_height;                 // eye level from floor
    fixed splash_height;                 // water splash trigger height

    // stereo 3D
    fixed half_camera_separation;        // eye separation for stereo
};
```

**Default values** (`physics_models.h:38-75`) - two models exist:

| Constant | Walking | Running |
|----------|---------|---------|
| `maximum_forward_velocity` | `FIXED_ONE/14` | `FIXED_ONE/8` |
| `maximum_backward_velocity` | `FIXED_ONE/17` | `FIXED_ONE/12` |
| `acceleration` | `FIXED_ONE/200` | `FIXED_ONE/100` |
| `gravitational_acceleration` | `FIXED_ONE/400` | `FIXED_ONE/400` |
| `terminal_velocity` | `FIXED_ONE/7` | `FIXED_ONE/7` |
| `angular_acceleration` | `5*FIXED_ONE/8` | `5*FIXED_ONE/4` |
| `maximum_angular_velocity` | `6*FIXED_ONE` | `10*FIXED_ONE` |
| `radius` | `FIXED_ONE/4` | `FIXED_ONE/4` |
| `height` | `4*FIXED_ONE/5` | `4*FIXED_ONE/5` |
| `camera_height` | `FIXED_ONE/5` | `FIXED_ONE/5` |

---

## Loading Physics Files

The engine loads physics at game start (`import_definitions.c:69-93`):

```c
void import_definition_structures(void) {
    struct wad_data *wad;
    boolean bungie_physics;

    wad = get_physics_wad_data(&bungie_physics);
    if (wad) {
        if (!bungie_physics && !warned_about_physics) {
            // warn user that external physics are active
            alert_user(infoError, strERRORS, warningExternalPhysicsModel, 0);
            warned_about_physics = TRUE;
        }

        import_physics_wad_data(wad);
        free_wad(wad);
    }
}
```

Each tagged chunk is copied directly into the corresponding array (`import_definitions.c:130-151`):

```c
void import_physics_wad_data(struct wad_data *wad) {
    for (index = 0; index < NUMBER_OF_DEFINITIONS; ++index) {
        struct definition_data *definition = definitions + index;
        void *data = extract_type_from_wad(wad, definition->tag, &length);
        if (data) {
            memcpy(definition->data, data, length);
        }
    }
}
```

---

## Creating Physics Patch Files

The `physics_patches.c` tool creates delta patches between two physics files. Instead of storing complete physics, patches store only the changed bytes.

**Patch file structure**:
```
┌─────────────────────────────────┐
│ WAD Header                      │
│   data_version = PHYSICS_DATA_VERSION
│   parent_checksum = original file checksum
├─────────────────────────────────┤
│ For each changed tag:           │
│   Tag + offset + changed bytes  │
├─────────────────────────────────┤
│ WAD Directory                   │
└─────────────────────────────────┘
```

**Patch creation algorithm** (`physics_patches.c:192-291`):
1. Load original and new physics WADs
2. For each tag, compare byte-by-byte
3. Find first non-matching offset and length of changes
4. Store only the delta: `(tag, offset, changed_bytes)`
5. Write patch with parent checksum for validation

---

## Network Physics Synchronization

In multiplayer, the host's physics are transmitted to all clients:

```c
// get physics as flat data for network transmission
void *get_network_physics_buffer(long *physics_length) {
    void *data = get_flat_data(&physics_file, FALSE, 0);
    if (data) {
        *physics_length = get_flat_data_length(data);
    }
    return data;
}

// client receives and applies physics
void process_network_physics_model(void *data) {
    struct wad_data *wad = inflate_flat_data(data, &header);
    if (wad) {
        import_physics_wad_data(wad);
        free_wad(wad);
    }
}
```

This ensures all players use identical physics for deterministic simulation.

---

## Byte Order Considerations

All physics data is stored in **big-endian** format (68K Mac native). When loading on little-endian systems (x86), byte swapping is required for all multi-byte fields:

```c
// example: swap a fixed-point value
fixed swap_fixed(fixed val) {
    return ((val & 0xFF) << 24) | ((val & 0xFF00) << 8) |
           ((val & 0xFF0000) >> 8) | ((val & 0xFF000000) >> 24);
}
```

---

## Common Modifications

| Modification | Tag | Fields to Change |
|--------------|-----|------------------|
| Player speed | `'PXpx'` | `maximum_forward_velocity`, `acceleration` |
| Gravity | `'PXpx'` | `gravitational_acceleration`, `terminal_velocity` |
| Jump height | `'PXpx'` | `climbing_acceleration` |
| Weapon damage | `'WPpx'` | Projectile reference → `'PRpx'` damage fields |
| Monster health | `'MNpx'` | `vitality` field |
| Monster speed | `'MNpx'` | `speed` field |

---

## See Also

- [Chapter 6: Physics Engine](06_physics.md) - How physics constants are used
- [Chapter 8: Entities](08_entities.md) - Monster and projectile behavior
- [Chapter 10: File Formats](10_file_formats.md) - WAD file structure
- [Appendix H: Film Format](appendix_h_film_format.md) - Recording/replay system
- [Appendix J: Modding Cookbook](appendix_j_cookbook.md) - Step-by-step modification examples
