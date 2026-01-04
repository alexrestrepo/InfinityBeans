# Chapter 34: Dynamic Lighting

## Light Animation and Polygon Shading

> **For Porting:** `lightsource.c` is fully portable! No Mac dependencies. The lighting system uses pure integer math and integrates with the rendering pipeline via shading table selection.

> **Source Files:** `lightsource.c` (424 lines), `lightsource.h` (119 lines)

---

## 34.1 What Problem Are We Solving?

Marathon needs dynamic lighting that can:

- **Animate lights** for flickering torches, pulsing machinery, and dramatic effects
- **Respond to triggers** via switches and platforms (tag system)
- **Create atmosphere** through varied lighting states
- **Support gameplay** with lights revealing or concealing areas

**The constraints:**
- Must be deterministic (critical for networking)
- Must integrate with the shading table system (no per-pixel lighting)
- Must support complex behaviors (multi-phase animations)
- Must be efficient (many lights updated per tick)

**Marathon's solution: State Machine with Lighting Functions**

Each light has a 6-state machine controlling its intensity over time. Four mathematical functions (constant, linear, smooth, flicker) define how intensity transitions between values.

---

## 34.2 Lighting Architecture

### How Lights Work

```
LIGHT CONCEPT:

Lights don't cast rays - they set SHADING TABLE selection for polygons.

Polygon with light_index = 5:
┌────────────────────────────────────────┐
│                                        │
│  When light 5 intensity = 100%:        │
│  → Polygon uses bright shading table   │
│  → Textures appear fully lit           │
│                                        │
│  When light 5 intensity = 25%:         │
│  → Polygon uses dark shading table     │
│  → Textures appear dimmed              │
│                                        │
└────────────────────────────────────────┘

Light intensity (0-65535) maps to shading table index.
```

### Light States

From `lightsource.h:18-26`, Marathon defines 6 light states:

| State | Name | Description |
|-------|------|-------------|
| 0 | `_light_becoming_active` | Transitioning from inactive to active |
| 1 | `_light_primary_active` | Main active state |
| 2 | `_light_secondary_active` | Secondary active animation |
| 3 | `_light_becoming_inactive` | Transitioning from active to inactive |
| 4 | `_light_primary_inactive` | Main inactive state |
| 5 | `_light_secondary_inactive` | Secondary inactive animation |

### Lighting Functions

From `lightsource.h:30-37`, four functions control intensity transitions:

| Function | Name | Behavior |
|----------|------|----------|
| 0 | `_constant_lighting_function` | Immediate jump to target |
| 1 | `_linear_lighting_function` | Linear interpolation |
| 2 | `_smooth_lighting_function` | Cosine interpolation (eased) |
| 3 | `_flicker_lighting_function` | Smooth + random variation |

---

## 34.3 Light Data Structures

### Static Light Data (Editor-Defined)

From `lightsource.h:39-63`:

```c
struct lighting_function_specification {  /* 14 bytes */
    short function;              // Which function (0-3)
    short period;                // Ticks for full cycle
    short delta_period;          // Random variation in period
    fixed intensity;             // Target intensity (0-65536)
    fixed delta_intensity;       // Random variation in intensity
};

struct static_light_data {  /* 100 bytes */
    short type;                  // Light type
    word flags;                  // Behavior flags
    short phase;                 // Initial phase offset (0-360)

    // Six lighting specifications (one per state)
    struct lighting_function_specification
        primary_active,          // State 1
        secondary_active,        // State 2
        becoming_active,         // State 0
        primary_inactive,        // State 4
        secondary_inactive,      // State 5
        becoming_inactive;       // State 3

    short tag;                   // For switch/platform triggers
};
```

### Dynamic Light Data (Runtime State)

From `lightsource.h:65-84`:

```c
struct light_data {  /* 128 bytes */
    word flags;                  // Runtime state flags
    short type;                  // Light type
    short mode;                  // Current state (0-5)
    short phase;                 // Current phase in cycle

    // Specifications copied from static data
    struct lighting_function_specification
        primary_active, secondary_active, becoming_active,
        primary_inactive, secondary_inactive, becoming_inactive;

    fixed intensity;             // Current light intensity

    short tag;                   // For triggers
    short unused[4];
};
```

---

## 34.4 Light State Machine

### State Transition Diagram

```
                    Light activated (tag switch/initial)
                                   │
                                   ▼
                    ┌──────────────────────────┐
           ┌───────│    BECOMING_ACTIVE (0)    │───────┐
           │       │   Transition to active    │       │
           │       └───────────┬───────────────┘       │
           │                   │                       │
           │         phase >= period                   │
           │                   │                       │
           │                   ▼                       │
           │       ┌──────────────────────────┐        │
           │       │    PRIMARY_ACTIVE (1)    │◄───────┤
           │       │   Main lit state         │        │
           │       └───────────┬───────────────┘       │
           │                   │                       │
           │         phase >= period                   │
           │         (if stateless)                    │
           │                   │                       │
           │                   ▼                       │
           │       ┌──────────────────────────┐        │
           │       │   SECONDARY_ACTIVE (2)   │────────┘
           │       │   Alternate lit state    │
           │       └──────────────────────────┘
           │
           │       Light deactivated (tag switch)
           │                   │
           │                   ▼
           │       ┌──────────────────────────┐
           └──────►│   BECOMING_INACTIVE (3)  │◄───────┐
                   │   Transition to dark     │        │
                   └───────────┬───────────────┘       │
                               │                       │
                     phase >= period                   │
                               │                       │
                               ▼                       │
                   ┌──────────────────────────┐        │
                   │   PRIMARY_INACTIVE (4)   │◄───────┤
                   │   Main dark state        │        │
                   └───��───────┬───────────────┘       │
                               │                       │
                     phase >= period                   │
                     (if stateless)                    │
                               │                       │
                               ▼                       │
                   ┌──────────────────────────┐        │
                   │  SECONDARY_INACTIVE (5)  │────────┘
                   │   Alternate dark state   │
                   └──────────────────────────┘
```

### State Machine Flags

From `lightsource.h:86-91`:

| Flag | Bit | Description |
|------|-----|-------------|
| `_light_is_initially_active` | 0 | Starts in active state |
| `_light_has_slaved_intensities` | 1 | Slave intensity to another light |
| `_light_is_stateless` | 2 | Cycles continuously (no trigger needed) |

---

## 34.5 Update Loop

From `lightsource.c:155-210`, the `update_lights()` function:

```c
void update_lights(void)
{
    for each light:
        struct light_data *light = get_light_data(i);
        struct lighting_function_specification *spec;

        // Get current state's specification
        switch (light->mode)
        {
            case _light_becoming_active:
                spec = &light->becoming_active;
                break;
            case _light_primary_active:
                spec = &light->primary_active;
                break;
            // ... other states
        }

        // Calculate current intensity using lighting function
        light->intensity = lighting_function_dispatch(
            spec->function,
            light->phase,
            spec->period,
            spec->intensity,
            spec->delta_intensity
        );

        // Advance phase
        light->phase++;

        // Check for state transition
        if (light->phase >= spec->period)
        {
            light->phase = 0;
            change_light_state(i, get_next_state(light->mode));
        }
}
```

---

## 34.6 Lighting Functions

The four lighting functions determine how intensity changes over time.

### Function Implementations

From `lightsource.c:384-423`:

```c
// _constant_lighting_function (0)
// Immediately returns target intensity
fixed constant_function(short phase, short period,
                        fixed intensity, fixed delta_intensity)
{
    (void)phase;
    (void)period;
    return intensity + random_intensity_adjustment(delta_intensity);
}

// _linear_lighting_function (1)
// Linearly interpolates from 0 to intensity
fixed linear_function(short phase, short period,
                      fixed intensity, fixed delta_intensity)
{
    fixed base = (intensity * phase) / period;
    return base + random_intensity_adjustment(delta_intensity);
}

// _smooth_lighting_function (2)
// Cosine interpolation (eased in/out)
fixed smooth_function(short phase, short period,
                      fixed intensity, fixed delta_intensity)
{
    // Convert phase to angle (0 to π)
    angle theta = (phase * HALF_CIRCLE) / period;

    // Cosine gives smooth ease: (1 - cos(θ)) / 2
    fixed base = intensity - FIXED_INTEGERAL_PART(
        intensity * cosine_table[theta] + FIXED_ONE_HALF);

    return base + random_intensity_adjustment(delta_intensity);
}

// _flicker_lighting_function (3)
// Smooth function plus random noise
fixed flicker_function(short phase, short period,
                       fixed intensity, fixed delta_intensity)
{
    fixed smooth = smooth_function(phase, period, intensity, 0);

    // Add random flicker
    fixed flicker = (global_random() * delta_intensity) >> 15;

    return smooth + flicker - (delta_intensity >> 1);
}
```

### Visual Representation

```
CONSTANT:               LINEAR:                 SMOOTH:                 FLICKER:
intensity               intensity               intensity               intensity
    │                       │                       │                       │
max ├─────────────      max ├            ╱      max ├          ╭───╮    max ├    ╱╲  ╱╲╱╲
    │                       │          ╱            │        ╱     ╲       │  ╱╲╱  ╲╱
    │                       │        ╱              │      ╱         ╲     │ ╱
    │                       │      ╱                │    ╱             ╲   │╱
min ├─                  min ├────╱              min ├──╯               ╰──min ├──
    └───────────► time      └───────────► time      └───────────► time      └───────────► time

    Immediate jump          Even ramp              Eased transition        Smooth + random
```

---

## 34.7 Tag System Integration

Lights can be controlled by switches and platforms via Marathon's tag system.

### Tag-Based Light Control

From `lightsource.c:227-260`:

```c
void set_tagged_light_statuses(short tag, boolean status)
{
    for (i = 0; i < dynamic_world->light_count; i++)
    {
        struct light_data *light = get_light_data(i);

        if (light->tag == tag)
        {
            set_light_status(i, status);
        }
    }
}

void set_light_status(short light_index, boolean status)
{
    struct light_data *light = get_light_data(light_index);

    if (status)
    {
        // Activate: transition to becoming_active
        if (light->mode >= _light_becoming_inactive)
        {
            change_light_state(light_index, _light_becoming_active);
        }
    }
    else
    {
        // Deactivate: transition to becoming_inactive
        if (light->mode < _light_becoming_inactive)
        {
            change_light_state(light_index, _light_becoming_inactive);
        }
    }
}
```

### Control Flow

```
Player activates switch with tag=5:
           │
           ▼
    change_panel_state()
           │
           ▼
    set_tagged_light_statuses(5, TRUE)
           │
           ▼
    For each light with tag=5:
           │
           ├─► set_light_status(light, TRUE)
           │           │
           │           ▼
           │   change_light_state(_light_becoming_active)
           │           │
           │           ▼
           │   Light begins transition animation
           │
           └─► (next light with tag=5)
```

---

## 34.8 Light-to-Polygon Mapping

Each polygon references a light index that determines its brightness.

### Polygon Light Assignment

From `map.h` (polygon structure):

```c
struct polygon_data {
    // ...
    short floor_lightsource_index;    // Light for floor
    short ceiling_lightsource_index;  // Light for ceiling
    // ...
};
```

### Intensity to Shading Table

During rendering, light intensity selects a shading table:

```c
// From render.c (conceptual)
void get_polygon_shading(short polygon_index)
{
    struct polygon_data *polygon = get_polygon_data(polygon_index);
    struct light_data *light = get_light_data(polygon->floor_lightsource_index);

    // Intensity (0-65535) maps to shading table index
    // Higher intensity = brighter shading table
    short shade_index = light->intensity >> SHADE_TABLE_SHIFT;

    return &shading_tables[shade_index];
}
```

### Visualization

```
Light intensity progression:

intensity = 65535 (100%)           intensity = 32768 (50%)           intensity = 0 (0%)
┌────────────────────────┐         ┌────────────────────────┐         ┌────────────────────────┐
│████████████████████████│         │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│         │░░░░░░░░░░░░░░░░░░░░░░░░│
│████ FULLY LIT █████████│         │▓▓▓▓ MEDIUM ▓▓▓▓▓▓▓▓▓▓▓│         │░░░░ DARK ░░░░░░░░░░░░░│
│████████████████████████│         │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│         │░░░░░░░░░░░░░░░░░░░░░░░░│
└────────────────────────┘         └────────────────────────┘         └────────────────────────┘
   Shading table 31                   Shading table 15                   Shading table 0
```

---

## 34.9 Light Types

Marathon defines several built-in light types with preset behaviors.

### Standard Light Types

| Type | Active Behavior | Inactive Behavior | Use Case |
|------|-----------------|-------------------|----------|
| Normal | Constant bright | Constant dark | Standard rooms |
| Strobe | Flicker bright | Constant dark | Alarms, machinery |
| Media | Smooth pulse | Smooth pulse | Underwater |
| Outdoor | Constant (sky) | Constant dark | Exterior areas |
| Lava | Flicker orange | Flicker dim | Volcanic areas |

### Type Definition Example

```c
// Conceptual - actual definitions in map editor
struct light_type_preset {
    struct lighting_function_specification active;
    struct lighting_function_specification inactive;
    word default_flags;
};

static struct light_type_preset strobe_light = {
    .active = {
        .function = _flicker_lighting_function,
        .period = 15,           // 0.5 seconds
        .delta_period = 5,
        .intensity = FIXED_ONE, // Full brightness
        .delta_intensity = FIXED_ONE/4
    },
    .inactive = {
        .function = _constant_lighting_function,
        .period = 1,
        .delta_period = 0,
        .intensity = FIXED_ONE/8,  // Dim
        .delta_intensity = 0
    },
    .default_flags = _light_is_stateless  // Continuous cycle
};
```

---

## 34.10 Stateless vs Triggered Lights

### Stateless Lights (`_light_is_stateless`)

Continuously cycle through all states without requiring activation:

```
Stateless light lifecycle:

    PRIMARY_ACTIVE ──► SECONDARY_ACTIVE ──► BECOMING_INACTIVE
           ▲                                        │
           │                                        ▼
    BECOMING_ACTIVE ◄── SECONDARY_INACTIVE ◄── PRIMARY_INACTIVE

    Loops forever without external trigger.
    Used for: ambient animations, always-flickering torches
```

### Triggered Lights

Only transition when activated/deactivated via tag:

```
Triggered light lifecycle:

    Inactive:                               Active:
    ┌─────────────────────────────┐         ┌─────────────────────────────┐
    │                             │         │                             │
    │  PRIMARY_INACTIVE ◄─────►   │   TAG   │   ◄─────► PRIMARY_ACTIVE    │
    │       │         ▲           │ ══════► │   ▲              │          │
    │       ▼         │           │         │   │              ▼          │
    │  SECONDARY_INACTIVE         │         │         SECONDARY_ACTIVE    │
    │                             │         │                             │
    └─────────────────────────────┘         └─────────────────────────────┘

    Stays in inactive/active group until tag changes state.
    Used for: switch-controlled lights, door lights
```

---

## 34.11 Initialization

From `lightsource.c:80-150`:

```c
void new_light(struct static_light_data *static_data)
{
    struct light_data *light = allocate_light();

    // Copy specifications from static data
    light->primary_active = static_data->primary_active;
    light->secondary_active = static_data->secondary_active;
    light->becoming_active = static_data->becoming_active;
    light->primary_inactive = static_data->primary_inactive;
    light->secondary_inactive = static_data->secondary_inactive;
    light->becoming_inactive = static_data->becoming_inactive;

    light->tag = static_data->tag;
    light->flags = static_data->flags;

    // Set initial state
    if (LIGHT_IS_INITIALLY_ACTIVE(light))
    {
        light->mode = _light_primary_active;
        light->intensity = light->primary_active.intensity;
    }
    else
    {
        light->mode = _light_primary_inactive;
        light->intensity = light->primary_inactive.intensity;
    }

    // Apply phase offset
    light->phase = static_data->phase;
}
```

---

## 34.12 Summary

### Key Concepts

- **Lights control polygon shading** — intensity maps to shading table selection
- **6-state machine** — active/inactive with primary, secondary, and transition states
- **4 lighting functions** — constant, linear, smooth, flicker
- **Tag system** — switches and platforms can control lights
- **Stateless option** — lights can cycle continuously without triggers

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAXIMUM_LIGHTS_PER_MAP` | 64 | Light limit per level |
| `NUMBER_OF_LIGHT_STATES` | 6 | State machine states |
| `NUMBER_OF_LIGHTING_FUNCTIONS` | 4 | Animation functions |
| `FIXED_ONE` | 65536 | Maximum intensity |

### Key Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `lightsource.c` | 424 | Light logic and update |
| `lightsource.h` | 119 | Data structures, enums |

### Source Reference Summary

| Function | Location | Purpose |
|----------|----------|---------|
| `new_light()` | lightsource.c:80 | Initialize light from map data |
| `update_lights()` | lightsource.c:155 | Main update loop |
| `set_light_status()` | lightsource.c:242 | Activate/deactivate light |
| `set_tagged_light_statuses()` | lightsource.c:227 | Tag-based control |
| `change_light_state()` | lightsource.c:286 | State machine transition |
| `lighting_function_dispatch()` | lightsource.c:384 | Calculate intensity |

---

*Next: [Chapter 35: Player Data](35_player.md) - Player state and inventory*

