# Chapter 27: Scenery Objects

## Static Decorations, Lights, and Destructibles

> **For Porting:** Scenery logic in `scenery.c` is fully portable. Scenery uses the standard object and shape systems.

---

## 27.1 What Problem Are We Solving?

Levels need visual detail beyond walls and floors:

- **Ambient objects** - Debris, blood pools, equipment
- **Light fixtures** - Lamps that can be destroyed
- **Animated decorations** - Machinery, water effects
- **Blocking objects** - Barrels, crates, pillars

---

## 27.2 Scenery Flags

From `scenery_definitions.h:10-14`:
```c
enum /* scenery flags */ {
    _scenery_is_solid         = 0x0001,  // Blocks movement
    _scenery_is_animated      = 0x0002,  // Has animation frames
    _scenery_can_be_destroyed = 0x0004   // Takes damage
};
```

---

## 27.3 Scenery Definition Structure

From `scenery_definitions.h:17-27`:
```c
struct scenery_definition {
    word flags;                    // Behavior flags
    shape_descriptor shape;        // Visual appearance

    world_distance radius;         // Collision radius
    world_distance height;         // Collision height (negative = hanging)

    short destroyed_effect;        // Effect when destroyed
    shape_descriptor destroyed_shape;  // Appearance after destruction
};
```

---

## 27.4 Scenery by Environment

### Water Environment (`_collection_scenery1`)

| Index | Description | Solid | Animated | Destructible |
|-------|-------------|-------|----------|--------------|
| 0 | Pistol clip | No | No | No |
| 1-2 | Lights | Yes | No | Yes |
| 3 | Siren | Yes | No | Yes (explosion) |
| 4-6 | Blood, puddles | No | No | No |
| 7 | Water animation | No | Yes | No |
| 8-9 | Supply cans | Yes | No | No |
| 10 | Machine | No | Yes | No |

### Lava Environment (`_collection_scenery2`)

| Index | Description | Solid | Animated | Destructible |
|-------|-------------|-------|----------|--------------|
| 0 | Light dirt | No | No | No |
| 1 | Dark dirt | No | No | No |
| 2-4 | Bones, skull | No | No | No |
| 5-6 | Hanging lights | Yes | No | Yes |
| 7-9 | Cylinders, blocks | Yes | No | No |

### Sewage Environment (`_collection_scenery3`)

| Index | Description | Solid | Animated | Destructible |
|-------|-------------|-------|----------|--------------|
| 0-1 | Green lights | Yes | No | Yes |
| 2-5 | Junk, antennas | No | No | No |
| 6 | Supply can | Yes | No | No |
| 7-10 | Bones, gore | No | No | No |

### Alien Environment (`_collection_scenery5`)

| Index | Description | Solid | Animated | Destructible |
|-------|-------------|-------|----------|--------------|
| 0-2 | Alien lights | Yes | No | Yes |
| 3-8 | Organic objects | No | No | No |
| 9 | Hunter shield | No | No | No |
| 10 | Alien sludge | No | No | No |

---

## 27.5 Creating Scenery

```c
short new_scenery(
    struct object_location *location,
    short scenery_type)
{
    struct scenery_definition *definition = get_scenery_definition(scenery_type);

    // Create map object with scenery shape
    short object_index = new_map_object(location, definition->shape);
    struct object_data *object = get_object_data(object_index);

    // Configure object
    SET_OBJECT_OWNER(object, _object_is_scenery);
    SET_OBJECT_SOLIDITY(object, definition->flags & _scenery_is_solid);
    object->permutation = scenery_type;  // Remember type for destruction

    return object_index;
}
```

---

## 27.6 Animated Scenery

```c
#define MAXIMUM_ANIMATED_SCENERY_OBJECTS 20

static short animated_scenery_object_count;
static short *animated_scenery_object_indexes;

void randomize_scenery_shapes(void) {
    animated_scenery_object_count = 0;

    for_each_object(object) {
        if (GET_OBJECT_OWNER(object) == _object_is_scenery) {
            struct scenery_definition *definition =
                get_scenery_definition(object->permutation);

            // Try to randomize starting frame
            if (!randomize_object_sequence(object_index, definition->shape)) {
                // If animated, add to update list
                if (animated_scenery_object_count < MAXIMUM_ANIMATED_SCENERY_OBJECTS) {
                    animated_scenery_object_indexes[animated_scenery_object_count++] =
                        object_index;
                }
            }
        }
    }
}

void animate_scenery(void) {
    // Called each tick from game loop
    for (short i = 0; i < animated_scenery_object_count; ++i) {
        animate_object(animated_scenery_object_indexes[i]);
    }
}
```

### Animation Limit

Only **20 scenery objects** can animate simultaneously. This is a performance constraint from the original hardware.

---

## 27.7 Destructible Scenery

```c
void damage_scenery(short object_index) {
    struct object_data *object = get_object_data(object_index);
    struct scenery_definition *definition =
        get_scenery_definition(object->permutation);

    if (definition->flags & _scenery_can_be_destroyed) {
        // Change to destroyed appearance
        object->shape = definition->destroyed_shape;

        // Spawn destruction effect (sparks, explosion)
        new_effect(&object->location, object->polygon,
            definition->destroyed_effect, object->facing);

        // No longer special (becomes normal debris)
        SET_OBJECT_OWNER(object, _object_is_normal);
    }
}
```

### Destruction Flow

```
┌──────────────────────┐     damage_scenery()      ┌──────────────────────┐
│   INTACT SCENERY     │ ────────────────────────► │   DESTROYED STATE    │
│                      │                           │                      │
│      ┌──┐            │                           │       ╲╱             │
│      │██│  Hanging   │                           │       ──   Broken    │
│      └──┘   Light    │                           │           Light      │
│                      │      + _effect_lamp_      │                      │
│ _scenery_can_be_     │        _breaking          │ _object_is_normal    │
│ _destroyed           │                           │                      │
└──────────────────────┘                           └──────────────────────┘
```

---

## 27.8 Scenery Dimensions

```c
void get_scenery_dimensions(
    short scenery_type,
    world_distance *radius,
    world_distance *height)
{
    struct scenery_definition *definition = get_scenery_definition(scenery_type);

    *radius = definition->radius;
    *height = definition->height;
}
```

### Height Convention

- **Positive height**: Floor-standing object
- **Negative height**: Ceiling-hanging object (lights, decorations)

```
Ceiling ────────────────────────────────────────────
              ┌─┐
              │█│  height = -WORLD_ONE/8 (hanging)
              └─┘

         ┌────────┐
         │████████│  height = WORLD_ONE_HALF (standing)
         │████████│
Floor ───┴────────┴─────────────────────────────────
```

---

## 27.9 Scenery vs Other Objects

| Property | Scenery | Monster | Item | Effect |
|----------|---------|---------|------|--------|
| **Solid** | Optional | Yes | No | No |
| **Animated** | Optional | Yes | No | Yes |
| **Destructible** | Optional | Yes | No | No |
| **AI** | No | Yes | No | No |
| **Pickup** | No | No | Yes | No |
| **Auto-remove** | No | On death | On pickup | On animation end |

---

## 27.10 Scenery API

```c
// Initialize scenery system
void initialize_scenery(void);

// Create new scenery object
short new_scenery(struct object_location *location, short scenery_type);

// Update animated scenery (called each tick)
void animate_scenery(void);

// Randomize starting animation frames
void randomize_scenery_shapes(void);

// Get collision dimensions
void get_scenery_dimensions(short scenery_type,
    world_distance *radius, world_distance *height);

// Apply damage to destructible scenery
void damage_scenery(short object_index);
```

---

## 27.11 Summary

Marathon's scenery system provides:

- **Environment-specific** decorations (5 collections)
- **Optional solidity** for blocking movement
- **Limited animation** (20 objects max)
- **Destruction system** with effects
- **Height flexibility** for floor/ceiling objects

### Key Source Files

| File | Purpose |
|------|---------|
| `scenery.c` | Scenery creation and updates |
| `scenery.h` | Scenery API |
| `scenery_definitions.h` | Scenery type data |

---

*Next: [Chapter 28: Computer Terminal System](28_terminals.md) - Story content and interactivity*
