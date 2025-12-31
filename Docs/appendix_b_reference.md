# Appendix B: Quick Reference

## Common Constants, Formulas, and Limits

---

## World Units

| Constant | Value | Meaning |
|----------|-------|---------|
| `WORLD_ONE` | 1024 | Base unit (≈1 meter) |
| `WORLD_ONE_HALF` | 512 | Half unit |
| `WORLD_ONE_FOURTH` | 256 | Quarter unit |
| `WORLD_FRACTIONAL_PART(x)` | `x & (WORLD_ONE-1)` | Fractional portion |
| `WORLD_INTEGERAL_PART(x)` | `x >> WORLD_FRACTIONAL_BITS` | Integer portion |

### Common Distances

| Object | Height/Size |
|--------|-------------|
| Player standing | 819 units |
| Player crouching | 409 units |
| Player radius | 256 units |
| Step-up height | 256 units |
| Maximum climb | 1024 units (with running jump) |
| Door height (typical) | 1024 units |

---

## Fixed-Point Math

| Constant | Value | Meaning |
|----------|-------|---------|
| `FIXED_ONE` | 0x10000 (65536) | 1.0 in fixed-point |
| `FIXED_ONE_HALF` | 0x8000 | 0.5 in fixed-point |
| `FIXED_FRACTIONAL_BITS` | 16 | Bits for fraction |

### Conversion Formulas

```c
// Integer to fixed
#define INTEGER_TO_FIXED(i) ((i) << FIXED_FRACTIONAL_BITS)

// Fixed to integer (truncate)
#define FIXED_INTEGERAL_PART(f) ((f) >> FIXED_FRACTIONAL_BITS)

// Fixed to integer (round)
#define FIXED_TO_INTEGER_ROUND(f) (((f) + FIXED_ONE_HALF) >> FIXED_FRACTIONAL_BITS)

// Fixed multiplication
#define FIXED_MULTIPLY(a, b) (((long)(a) * (long)(b)) >> FIXED_FRACTIONAL_BITS)

// Float to fixed
#define FLOAT_TO_FIXED(f) ((fixed)((f) * FIXED_ONE))
```

---

## Type Mapping (Marathon → stdint.h)

For porting to modern platforms. See Chapter 24 for full details.

| Marathon Type | Bits | stdint.h | Signed? |
|---------------|------|----------|---------|
| `byte` | 8 | `uint8_t` | No |
| `word` | 16 | `uint16_t` | No |
| `boolean` | 8 | `uint8_t` | No |
| `short` | 16 | `int16_t` | Yes |
| `long` | 32 | `int32_t` | Yes |
| `fixed` | 32 | `int32_t` | Yes |
| `unsigned short` | 16 | `uint16_t` | No |
| `unsigned long` | 32 | `uint32_t` | No |

---

## Angles

| Constant | Value | Meaning |
|----------|-------|---------|
| `FULL_CIRCLE` | 512 | Complete rotation |
| `QUARTER_CIRCLE` | 128 | 90 degrees |
| `HALF_CIRCLE` | 256 | 180 degrees |
| `EIGHTH_CIRCLE` | 64 | 45 degrees |
| `SIXTEENTH_CIRCLE` | 32 | 22.5 degrees |

### Angle Conversion

```c
// Degrees to Marathon angle
angle = (degrees * 512) / 360;

// Marathon angle to degrees
degrees = (angle * 360) / 512;

// Marathon angle to radians
radians = (angle * 2 * PI) / 512;

// Normalize to 0-511 range
#define NORMALIZE_ANGLE(a) ((a) & (FULL_CIRCLE - 1))
```

---

## Timing

| Constant | Value | Meaning |
|----------|-------|---------|
| `TICKS_PER_SECOND` | 30 | Game simulation rate |
| `MACINTOSH_TICKS_PER_SECOND` | 60 | Mac OS timer rate |
| Tick duration | 33.33ms | Per game tick |

### Time Conversion

```c
// Seconds to ticks
ticks = seconds * TICKS_PER_SECOND;

// Ticks to milliseconds
ms = ticks * (1000 / TICKS_PER_SECOND);
```

---

## Array Limits

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAXIMUM_POLYGONS_PER_MAP` | 1024 | Map polygons |
| `MAXIMUM_LINES_PER_MAP` | 4096 | Map lines |
| `MAXIMUM_ENDPOINTS_PER_MAP` | 8192 | Map vertices |
| `MAXIMUM_SIDES_PER_MAP` | 4096 | Wall surfaces |
| `MAXIMUM_OBJECTS_PER_MAP` | 384 | All objects |
| `MAXIMUM_MONSTERS_PER_MAP` | 220 | Monster limit |
| `MAXIMUM_PROJECTILES_PER_MAP` | 32 | Active projectiles |
| `MAXIMUM_EFFECTS_PER_MAP` | 64 | Visual effects |
| `MAXIMUM_LIGHTS_PER_MAP` | 64 | Light sources |
| `MAXIMUM_PLATFORMS_PER_MAP` | 64 | Moving platforms |
| `MAXIMUM_MEDIAS_PER_MAP` | 16 | Liquid volumes |
| `MAXIMUM_CONTROL_PANELS_PER_MAP` | 64 | Switches/terminals |
| `MAXIMUM_COLLECTIONS` | 32 | Shape collections |
| `MAXIMUM_ANIMATED_SCENERY_OBJECTS` | 20 | Animated scenery |

---

## Rendering

| Constant | Value | Purpose |
|----------|-------|---------|
| `NORMAL_FIELD_OF_VIEW` | 80° | Standard FOV |
| `EXTRAVISION_FIELD_OF_VIEW` | 130° | Fisheye powerup |
| `POLYGON_QUEUE_SIZE` | 256 | Render tree nodes |
| `RENDER_FLAGS_BUFFER_SIZE` | 8192 | Polygon flags |
| `number_of_shading_tables` | 32 | Light levels |
| `shading_table_size` | 256 | Colors per table |

### View Limits

| Limit | Value |
|-------|-------|
| Maximum pitch up | +22.5° (1/16 circle) |
| Maximum pitch down | -22.5° (1/16 circle) |
| Maximum render depth | ~30,000 units |

---

## Physics

| Constant | Value | Purpose |
|----------|-------|---------|
| `PLAYER_MAXIMUM_FORWARD_VELOCITY` | 224 | Run speed |
| `PLAYER_MAXIMUM_BACKWARD_VELOCITY` | 179 | Back speed |
| `PLAYER_MAXIMUM_PERPENDICULAR_VELOCITY` | 179 | Strafe speed |
| `ANGULAR_VELOCITY` | 24 | Turn speed |
| `PLAYER_MAXIMUM_ELEVATION_VELOCITY` | 12 | Look speed |
| `PLAYER_MAXIMUM_ELEVATION_RANGE` | ±1/16 circle | Pitch range |
| `NORMAL_GRAVITY` | -5 | Downward accel |
| `NORMAL_TERMINAL_VELOCITY` | -100 | Max fall speed |

### Swimming (halved values)

| Parameter | Value |
|-----------|-------|
| Gravity | -2.5 |
| Terminal velocity | -50 |
| Movement speed | ×0.5 |

---

## Damage Types

| Type | ID | Description |
|------|-----|-------------|
| `_damage_explosion` | 0 | Grenade/rocket |
| `_damage_electrical_staff` | 1 | Staff weapon |
| `_damage_projectile` | 2 | Bullet/pellet |
| `_damage_absorbed` | 3 | Shield absorbed |
| `_damage_flame` | 4 | Fire damage |
| `_damage_hound_claws` | 5 | Melee attack |
| `_damage_alien_projectile` | 6 | Alien weapons |
| `_damage_hulk_slap` | 7 | Hulk melee |
| `_damage_compiler_bolt` | 8 | Energy bolt |
| `_damage_fusion_bolt` | 9 | Fusion pistol |
| `_damage_hunter_bolt` | 10 | Hunter attack |
| `_damage_fist` | 11 | Player punch |
| `_damage_teleporter` | 12 | Telefrag |
| `_damage_defender` | 13 | Drone attack |
| `_damage_yeti_claws` | 14 | F'lickta melee |
| `_damage_yeti_projectile` | 15 | F'lickta spit |
| `_damage_crushing` | 16 | Platform crush |
| `_damage_lava` | 17 | Lava contact |
| `_damage_suffocation` | 18 | Drowning |
| `_damage_goo` | 19 | Goo contact |
| `_damage_energy_drain` | 20 | Shield drain |
| `_damage_oxygen_drain` | 21 | Oxygen drain |
| `_damage_hummer_bolt` | 22 | Cyborg attack |
| `_damage_shotgun_projectile` | 23 | Shotgun pellet |

---

## Monster Classes

| Class | ID | Examples |
|-------|-----|----------|
| `_class_player` | 0 | Marine |
| `_class_human_civilian` | 1 | BOBs |
| `_class_madd` | 2 | MA-75 enemies |
| `_class_possessed_hummer` | 3 | Possessed drones |
| `_class_defender` | 4 | Drones |
| `_class_fighter` | 5 | Pfhor fighters |
| `_class_trooper` | 6 | Pfhor troopers |
| `_class_hunter` | 7 | Hunters |
| `_class_enforcer` | 8 | Enforcers |
| `_class_juggernaut` | 9 | Juggernauts |
| `_class_hummer` | 10 | Cyborgs |
| `_class_compiler` | 11 | Compilers |
| `_class_cyborg` | 12 | S'pht |
| `_class_assimilated_civilian` | 13 | Simulacrum |
| `_class_tick` | 14 | Ticks |
| `_class_yeti` | 15 | F'lickta |

---

## Color Conversion

```c
// Marathon uses 16-bit RGB555 colors internally
struct rgb_color {
    uint16 red, green, blue;  // 0-65535 range
};

// Convert to 32-bit ARGB (for modern displays)
uint32_t marathon_to_argb(struct rgb_color *c) {
    uint8_t r = c->red >> 8;
    uint8_t g = c->green >> 8;
    uint8_t b = c->blue >> 8;
    return 0xFF000000 | (r << 16) | (g << 8) | b;
}

// Convert from 8-bit to 32-bit
uint32_t pixel8_to_argb(uint8_t index, uint32_t *palette) {
    return palette[index];
}
```

---

## Byte Swapping (for x86/ARM)

```c
// Marathon data is big-endian
uint16_t swap16(uint16_t val) {
    return (val << 8) | (val >> 8);
}

uint32_t swap32(uint32_t val) {
    return ((val & 0xFF) << 24) |
           ((val & 0xFF00) << 8) |
           ((val & 0xFF0000) >> 8) |
           ((val & 0xFF000000) >> 24);
}

// Or use compiler builtins
#define swap16(x) __builtin_bswap16(x)
#define swap32(x) __builtin_bswap32(x)
```

---

## Common Flag Patterns

```c
// Flag macros from cseries.h
#define FLAG(bit) (1 << (bit))
#define TEST_FLAG16(flags, bit) ((flags) & FLAG(bit))
#define SET_FLAG16(flags, bit, value) \
    ((value) ? ((flags) |= FLAG(bit)) : ((flags) &= ~FLAG(bit)))

// Slot management
#define SLOT_IS_USED(s) ((s)->flags & _slot_used_flag)
#define MARK_SLOT_AS_USED(s) ((s)->flags |= _slot_used_flag)
#define MARK_SLOT_AS_FREE(s) ((s)->flags &= ~_slot_used_flag)
```

---

## WAD Tag Codes

| Tag | Description |
|-----|-------------|
| `'PNTS'` | Endpoints |
| `'LINS'` | Lines |
| `'POLY'` | Polygons |
| `'SIDS'` | Sides |
| `'LITE'` | Lights |
| `'OBJS'` | Objects |
| `'PLAT'` | Platforms |
| `'MEDI'` | Media |
| `'TERM'` | Terminal text |
| `'AMBI'` | Ambient sounds |
| `'rand'` | Random sounds |
| `'NOTE'` | Level notes |
| `'NAME'` | Level name |

---

*Next: [Appendix C: Source File Index](appendix_c_files.md) - File-by-file reference*
