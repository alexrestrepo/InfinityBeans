# Chapter 12: Data Structures Appendix

## Complete Type Reference

> **For Porting:** All types are portable. Replace Mac-specific `short`/`long` with explicit width types (`int16_t`/`int32_t`) for safety. The key insight is Marathon uses fixed-point extensively—`fixed` and `world_distance` are just integers with implicit decimal points.

---

## 12.1 Core Numeric Types

Marathon defines custom types for clarity and portability.

### Basic Types

```c
typedef long fixed;                  // 16.16 fixed-point
typedef short world_distance;        // 10 fractional bits
typedef unsigned short word;         // 16-bit unsigned
typedef unsigned char byte;          // 8-bit unsigned
typedef byte boolean;                // TRUE/FALSE
typedef short angle;                 // 512 angles per circle
```

### Modern Equivalents

| Marathon Type | Modern Type | Size | Notes |
|---------------|-------------|------|-------|
| `byte` | `uint8_t` | 1 | Unsigned byte |
| `word` | `uint16_t` | 2 | Unsigned 16-bit |
| `short` | `int16_t` | 2 | Signed 16-bit |
| `long` | `int32_t` | 4 | Signed 32-bit |
| `fixed` | `int32_t` | 4 | 16.16 fixed-point |
| `world_distance` | `int16_t` | 2 | 10-bit fractional |
| `angle` | `int16_t` | 2 | 512 per circle |

---

## 12.2 Coordinate Types

### Point Types

```c
struct world_point2d {
    world_distance x, y;
};

struct world_point3d {
    world_distance x, y, z;
};

struct fixed_point3d {
    fixed x, y, z;
};
```

### Vector Types

```c
struct world_vector2d {
    world_distance i, j;
};

struct world_vector3d {
    world_distance i, j, k;
};

struct fixed_vector3d {
    fixed i, j, k;
};
```

---

## 12.3 Key Constants

### Fixed-Point Constants

```c
#define FIXED_ONE (1<<16)               // 65536 = 1.0
#define FIXED_FRACTIONAL_BITS 16        // Bits after decimal
#define FIXED_ONE_HALF (1<<15)          // 0.5
```

### World Unit Constants

```c
#define WORLD_ONE 1024                  // 1 game unit
#define WORLD_FRACTIONAL_BITS 10        // 10 bits fractional
#define WORLD_ONE_HALF (WORLD_ONE/2)    // 512
#define WORLD_ONE_FOURTH (WORLD_ONE/4)  // 256
```

### Timing Constants

```c
#define TICKS_PER_SECOND 30             // Game logic rate
#define MACHINE_TICKS_PER_SECOND 60     // Mac VBL rate
```

### Angle Constants

```c
#define NUMBER_OF_ANGLES 512            // Angles in circle
#define FULL_CIRCLE 512
#define HALF_CIRCLE 256
#define QUARTER_CIRCLE 128
#define EIGHTH_CIRCLE 64
#define SIXTEENTH_CIRCLE 32
```

### Trigonometry Constants

```c
#define TRIG_SHIFT 10                   // Trig table precision
#define TRIG_MAGNITUDE (1<<TRIG_SHIFT)  // 1024
```

### Rendering Constants

```c
#define MAXIMUM_NODES 512               // Render tree nodes
#define MAXIMUM_SORTED_NODES 128        // Sorted render objects
#define MAXIMUM_RENDER_OBJECTS 72       // Sprites per frame
#define MAXIMUM_CLIPPING_WINDOWS 256    // Clipping regions
#define NORMAL_FIELD_OF_VIEW 80         // Default FOV degrees
```

### Physics Constants

```c
#define GRAVITATIONAL_ACCELERATION (FIXED_ONE/400)  // ~164
#define TERMINAL_VELOCITY (FIXED_ONE/7)             // ~9362
#define COEFFICIENT_OF_ABSORBTION 2                 // Bounce damping
```

---

## 12.4 Structure Sizes

| Structure | Size (bytes) | Max Count |
|-----------|--------------|-----------|
| `world_point2d` | 4 | Variable |
| `line_data` | 32 | 1024 |
| `side_data` | 64 | 2048 |
| `polygon_data` | 128 | 512 |
| `platform_data` | Variable | 256 |
| `media_data` | Variable | 32 |
| `light_data` | Variable | 256 |
| `monster_definition` | 128 | 47 types |
| `weapon_definition` | 196 | ~10 types |
| `projectile_definition` | 54 | ~40 types |
| `effect_data` | 16 | 64 |

---

## 12.5 Map Geometry Structures

### Endpoint (Vertex)

```c
// 4 bytes in file
struct endpoint_data {
    word flags;                  // Endpoint flags
    world_distance highest_adjacent_floor_height;
    world_distance lowest_adjacent_ceiling_height;
    world_point2d vertex;        // X, Y coordinates
    world_point2d transformed;   // After camera transform
    short supporting_polygon_index;
};
```

### Line Definition

```c
// 32 bytes
struct line_data {
    short endpoint_indexes[2];   // Start/end vertices
    word flags;                  // Solid, transparent, etc.
    short length;                // Pre-calculated length
    world_distance highest_adjacent_floor;
    world_distance lowest_adjacent_ceiling;
    short clockwise_polygon_side_index;
    short counterclockwise_polygon_side_index;
    short clockwise_polygon_owner;
    short counterclockwise_polygon_owner;
};
```

### Side Definition

```c
// 64 bytes
struct side_data {
    short type;                  // Full, high, low, composite, split
    word flags;
    struct side_texture_definition primary_texture;
    struct side_texture_definition secondary_texture;
    struct side_texture_definition transparent_texture;
    short polygon_index;
    short line_index;
    world_distance primary_transfer_mode;
    // ... additional fields
};
```

### Polygon Definition

```c
// 128 bytes
struct polygon_data {
    short type;                  // Normal, platform, minor ouch, etc.
    word flags;
    short permutation;

    short vertex_count;
    short endpoint_indexes[8];
    short line_indexes[8];

    world_distance floor_height;
    world_distance ceiling_height;

    short floor_texture;
    short ceiling_texture;
    short floor_transfer_mode;
    short ceiling_transfer_mode;

    short adjacent_polygon_indexes[8];

    short first_object;          // Linked list head
    short first_exclusion_zone_index;
    short line_exclusion_zone_count;
    short point_exclusion_zone_count;

    short floor_lightsource_index;
    short ceiling_lightsource_index;

    long area;                   // Pre-calculated area

    world_point2d center;        // Pre-calculated center

    short media_index;
    short ambient_sound_image_index;
    short random_sound_image_index;
};
```

---

## 12.6 Entity Structures

### Object Data

```c
struct object_data {
    short type;                  // Monster, scenery, item, etc.
    word flags;
    short polygon;
    world_point3d location;
    angle facing;
    shape_descriptor shape;
    short sequence;
    short next_object;           // Linked list
    short parasitic_object;
    // ... additional fields
};
```

### Monster Definition

```c
// 128 bytes
struct monster_definition {
    short collection;            // Sprite set
    short vitality;              // Hit points

    unsigned long immunities;    // Damage types ignored
    unsigned long weaknesses;    // Amplified damage
    unsigned long flags;

    long monster_class, friends, enemies;  // Faction relationships

    world_distance radius, height;
    world_distance visual_range, dark_visual_range;
    short half_visual_arc, half_vertical_visual_arc;

    short intelligence;          // Pathfinding quality
    short speed, gravity, terminal_velocity;

    short attack_frequency;
    struct attack_definition melee_attack, ranged_attack;
};
```

### Projectile Definition

```c
// 54 bytes
struct projectile_definition {
    short collection, shape;
    short detonation_effect, media_detonation_effect;
    short contrail_effect;
    short ticks_between_contrails;

    world_distance radius;       // Hit radius
    world_distance area_of_effect;
    struct damage_definition damage;

    unsigned long flags;

    world_distance speed;
    world_distance maximum_range;
};
```

### Weapon Definition

```c
// 196 bytes
struct weapon_definition {
    short weapon_class, flags;

    fixed idle_height, bob_amplitude, kick_height, reload_height;

    short collection;
    short idle_shape, firing_shape, reloading_shape;

    short ready_ticks, await_reload_ticks, loading_ticks;

    struct trigger_definition weapons_by_trigger[2];  // Primary/secondary
};

struct trigger_definition {
    short rounds_per_magazine;
    short ammunition_type;
    short ticks_per_round;       // Fire rate
    short recovery_ticks;
    short projectile_type;
    short theta_error;           // Spread angle
    short burst_count;           // Pellets per shot
};
```

---

## 12.7 Physics Structures

### Physics Variables

```c
struct physics_variables {
    fixed head_direction;        // Free look horizontal
    fixed direction;             // Heading (yaw)
    fixed elevation;             // Pitch
    fixed angular_velocity;
    fixed vertical_angular_velocity;

    fixed velocity;              // Forward/back speed
    fixed perpendicular_velocity;// Strafe speed

    fixed_point3d position;
    fixed_point3d last_position;

    fixed floor_height, ceiling_height;
    fixed media_height;
    fixed actual_height;

    fixed_vector3d external_velocity;
    fixed external_angular_velocity;

    fixed step_phase;
    fixed step_amplitude;

    word flags;
    short action;
};
```

### Physics Constants

```c
struct physics_constants {
    fixed maximum_forward_velocity;
    fixed maximum_backward_velocity;
    fixed maximum_perpendicular_velocity;

    fixed acceleration;
    fixed deceleration;
    fixed airborne_deceleration;
    fixed gravitational_acceleration;
    fixed climbing_acceleration;
    fixed terminal_velocity;
    fixed external_deceleration;

    fixed angular_acceleration;
    fixed angular_deceleration;
    fixed maximum_angular_velocity;
    fixed angular_recentering_velocity;
    fixed fast_angular_velocity;
    fixed fast_angular_maximum;
    fixed maximum_elevation;
    fixed external_angular_deceleration;

    fixed step_delta;
    fixed step_amplitude;

    fixed radius;
    fixed height;
    fixed dead_height;
    fixed camera_height;
    fixed splash_height;
    fixed half_camera_separation;
};
```

---

## 12.8 Damage Structure

```c
struct damage_definition {
    short type;      // Damage type (explosion, projectile, etc.)
    short flags;     // Modifier flags

    short base;      // Base damage amount
    short random;    // Random additional damage [0, random)
    fixed scale;     // Multiplier (FIXED_ONE = 1.0)
};
```

---

## 12.9 Rendering Structures

### Clipping Window

```c
struct clipping_window_data {
    short x0, x1;                // Horizontal bounds
    short y0, y1;                // Vertical bounds
    short left, right;           // Additional clipping
    short top, bottom;
    struct clipping_window_data* next;
};
```

### Render Node

```c
struct node_data {
    word flags;
    short polygon_index;
    struct clipping_window_data* clipping_windows;
};
```

---

## 12.10 Summary

Marathon's type system is straightforward:

**Numeric Types:**
- `fixed` = 32-bit signed, 16.16 format
- `world_distance` = 16-bit signed, 10-bit fractional
- `angle` = 16-bit, 512 per circle

**Coordinate Types:**
- 2D points and vectors for map geometry
- 3D points and vectors for world positions
- Fixed-point variants for high-precision physics

**For Porting:**
- Replace Mac types with `stdint.h` types
- Keep fixed-point logic as-is (it's critical for determinism)
- Structure sizes are important for file I/O

### Quick Reference

| Concept | Type | Value of 1.0 |
|---------|------|--------------|
| Fixed-point | `fixed` | 65536 |
| World units | `world_distance` | 1024 |
| Angles | `angle` | 512 (full circle) |

---

*Next: [Chapter 13: Sound System](13_sound.md) - 3D audio, channels, and ambient sounds*
