# Chapter 26: Visual Effects System

## Explosions, Splashes, Blood, and Particles

> **For Porting:** The effects system in `effects.c` and `effects.h` is fully portable. Effects use the standard shape/animation system for rendering.

---

## 26.1 What Problem Are We Solving?

Combat and environmental interactions need visual feedback:

- **Explosions** - Rockets, grenades, fusion bolts
- **Projectile trails** - Contrails behind missiles
- **Blood splashes** - Monster-specific colors
- **Liquid splashes** - Water, lava, goo impacts
- **Teleportation** - Object materialize/dematerialize
- **Spark effects** - Energy weapon impacts

---

## 26.2 Effect Types

```c
enum /* effect types */ {
    // Weapon explosions
    _effect_rocket_explosion,
    _effect_rocket_contrail,
    _effect_grenade_explosion,
    _effect_grenade_contrail,
    _effect_bullet_ricochet,
    _effect_alien_weapon_ricochet,
    _effect_flamethrower_burst,

    // Blood splashes (per monster type)
    _effect_fighter_blood_splash,
    _effect_player_blood_splash,
    _effect_civilian_blood_splash,
    _effect_assimilated_civilian_blood_splash,
    _effect_enforcer_blood_splash,
    _effect_trooper_blood_splash,
    _effect_cyborg_blood_splash,
    _effect_sewage_yeti_blood_splash,
    _effect_water_yeti_blood_splash,
    _effect_lava_yeti_blood_splash,
    _effect_vacuum_civilian_blood_splash,

    // Energy weapon effects
    _effect_compiler_bolt_minor_detonation,
    _effect_compiler_bolt_major_detonation,
    _effect_compiler_bolt_major_contrail,
    _effect_minor_fusion_detonation,
    _effect_major_fusion_detonation,
    _effect_major_fusion_contrail,
    _effect_minor_fusion_dispersal,
    _effect_major_fusion_dispersal,
    _effect_overloaded_fusion_dispersal,

    // Monster-specific
    _effect_fighter_projectile_detonation,
    _effect_fighter_melee_detonation,
    _effect_hunter_projectile_detonation,
    _effect_hunter_spark,
    _effect_minor_defender_detonation,
    _effect_major_defender_detonation,
    _effect_defender_spark,
    _effect_minor_hummer_projectile_detonation,
    _effect_major_hummer_projectile_detonation,
    _effect_durandal_hummer_projectile_detonation,
    _effect_hummer_spark,
    _effect_cyborg_projectile_detonation,
    _effect_juggernaut_spark,
    _effect_juggernaut_missile_contrail,
    _effect_yeti_melee_detonation,
    _effect_sewage_yeti_projectile_detonation,
    _effect_lava_yeti_projectile_detonation,

    // Teleportation
    _effect_teleport_object_in,
    _effect_teleport_object_out,

    // Media splashes (5 types × 4 sizes = 20 effects)
    _effect_small_water_splash,
    _effect_medium_water_splash,
    _effect_large_water_splash,
    _effect_large_water_emergence,
    // ... lava, goo, sewage, jjaro variants

    // Destruction
    _effect_water_lamp_breaking,
    _effect_lava_lamp_breaking,
    _effect_sewage_lamp_breaking,
    _effect_alien_lamp_breaking,
    _effect_metallic_clang,
    _effect_fist_detonation,

    NUMBER_OF_EFFECT_TYPES  // ~64
};

#define MAXIMUM_EFFECTS_PER_MAP 64
```

---

## 26.3 Effect Data Structure

```c
struct effect_data {  // 16 bytes
    short type;           // Effect type enum
    short object_index;   // Associated map object

    word flags;           // [slot_used.1] [unused.15]

    short data;           // Extra data (e.g., twin object for teleport)
    short delay;          // Ticks before effect becomes visible

    short unused[11];
};
```

---

## 26.4 Effect Definition

```c
enum /* effect flags */ {
    _end_when_animation_loops          = 0x0001,
    _end_when_transfer_animation_loops = 0x0002,
    _sound_only                        = 0x0004,  // No visual, just sound
    _make_twin_visible                 = 0x0008,  // Show linked object when done
    _media_effect                      = 0x0010   // Associated with liquid
};

struct effect_definition {
    short collection, shape;  // Visual source
    fixed sound_pitch;        // Audio pitch modifier
    word flags;               // Behavior flags
    short delay;              // Random delay range (ticks)
    short delay_sound;        // Sound when delay ends
};
```

---

## 26.5 Effect Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    EFFECT LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  new_effect()                                                    │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────┐                                             │
│  │  CREATION       │  Allocate effect slot                       │
│  │  • Set type     │  Create map object                          │
│  │  • Set position │  Configure animation                        │
│  │  • Set delay    │  May be invisible initially                 │
│  └────────┬────────┘                                             │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐                                             │
│  │  DELAY PHASE    │  (if delay > 0)                             │
│  │  • Invisible    │  Countdown each tick                        │
│  │  • Waiting      │  Play delay_sound when delay reaches 0      │
│  └────────┬────────┘                                             │
│           │ delay == 0                                           │
│           ▼                                                      │
│  ┌─────────────────┐                                             │
│  │  ANIMATION      │  Object becomes visible                     │
│  │  • Visible      │  animate_object() each tick                 │
│  │  • Animating    │  Play through shape frames                  │
│  └────────┬────────┘                                             │
│           │ last frame reached                                   │
│           ▼                                                      │
│  ┌─────────────────┐                                             │
│  │  REMOVAL        │  remove_effect()                            │
│  │  • Delete object│  Free effect slot                           │
│  │  • Free slot    │  May reveal twin object                     │
│  └─────────────────┘                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 26.6 Creating Effects

```c
short new_effect(
    world_point3d *origin,     // World position
    short polygon_index,       // Containing polygon
    short type,                // Effect type enum
    angle facing)              // Direction (for directional effects)
{
    struct effect_definition *definition = get_effect_definition(type);

    // Sound-only effects: just play sound, no object
    if (definition->flags & _sound_only) {
        play_world_sound(polygon_index, origin,
            get_shape_animation_sound(definition->collection, definition->shape));
        return NONE;
    }

    // Find free effect slot
    short effect_index = find_free_effect_slot();
    if (effect_index == NONE) return NONE;

    struct effect_data *effect = get_effect_data(effect_index);

    // Create map object for visual effect
    short object_index = new_map_object3d(origin, polygon_index,
        BUILD_DESCRIPTOR(definition->collection, definition->shape), facing);

    // Set up effect tracking
    MARK_SLOT_AS_USED(effect);
    effect->type = type;
    effect->object_index = object_index;
    effect->delay = definition->delay ? random() % definition->delay : 0;

    // Initially invisible if delayed
    if (effect->delay) {
        SET_OBJECT_INVISIBILITY(get_object_data(object_index), TRUE);
    }

    return effect_index;
}
```

---

## 26.7 Updating Effects

```c
void update_effects(void) {  // Called every tick
    for (short i = 0; i < MAXIMUM_EFFECTS_PER_MAP; ++i) {
        struct effect_data *effect = get_effect_data(i);

        if (SLOT_IS_USED(effect)) {
            struct effect_definition *definition = get_effect_definition(effect->type);
            struct object_data *object = get_object_data(effect->object_index);

            if (effect->delay) {
                // Delayed effect: count down
                if (!(--effect->delay)) {
                    // Delay complete - become visible
                    SET_OBJECT_INVISIBILITY(object, FALSE);
                    play_object_sound(effect->object_index, definition->delay_sound);
                }
            } else {
                // Active effect: animate
                word animation_flags = animate_object(effect->object_index);

                // Check for termination
                if ((animation_flags & _obj_last_frame_animated) &&
                    (definition->flags & _end_when_animation_loops)) {

                    // Make twin visible if needed (teleport-in)
                    if (definition->flags & _make_twin_visible) {
                        SET_OBJECT_INVISIBILITY(
                            get_object_data(effect->data), FALSE);
                    }

                    remove_effect(i);
                }
            }
        }
    }
}
```

---

## 26.8 Teleportation Effects

Special two-phase effects for object teleportation:

```c
void teleport_object_out(short object_index) {
    struct object_data *object = get_object_data(object_index);

    // Create fade-out effect at object location
    short effect_index = new_effect(&object->location, object->polygon,
        _effect_teleport_object_out, object->facing);

    struct effect_data *effect = get_effect_data(effect_index);
    struct object_data *effect_object = get_object_data(effect->object_index);

    // Effect copies object appearance
    effect_object->shape = object->shape;
    effect_object->transfer_mode = _xfer_fold_out;  // Shrinking effect
    effect_object->transfer_period = TELEPORTING_MIDPOINT;

    // Hide original object
    SET_OBJECT_INVISIBILITY(object, TRUE);

    play_object_sound(effect->object_index, _snd_teleport_out);
}

void teleport_object_in(short object_index) {
    struct object_data *object = get_object_data(object_index);

    short effect_index = new_effect(&object->location, object->polygon,
        _effect_teleport_object_in, object->facing);

    struct effect_data *effect = get_effect_data(effect_index);
    struct object_data *effect_object = get_object_data(effect->object_index);

    effect->data = object_index;  // Remember which object to reveal
    effect_object->transfer_mode = _xfer_fold_in;  // Growing effect

    // Object stays invisible until effect completes
    // (_make_twin_visible flag handles reveal)
}
```

### Teleport Visualization

```
TELEPORT OUT:                        TELEPORT IN:

    ██████████                             ░
   ████████████                           ░░░
  ██████████████      Shrink      Sparkle ░░░░░      Grow
   ████████████    ──────────►    ──────►  ░░░  ──────────►
    ██████████          ████               ░
                          ██
    Object            (fold_out)        effect         Object
    visible                                            visible

Timeline:
  0────────5────────10────────15────────20────────25
  │        │         │         │         │         │
  Out start│    Out done       In start  │     In done
           │         │         │         │
         Object hidden        Effect plays
```

---

## 26.9 Effect Categories

| Category | Examples | Behavior |
|----------|----------|----------|
| **Explosions** | Rocket, grenade, fusion | Play once, auto-remove |
| **Contrails** | Rocket trail, fusion trail | Spawn behind projectile |
| **Blood** | Per-monster type | Color-coded to monster |
| **Splashes** | Per-media type | Size varies by impact |
| **Sparks** | Hunter, defender, juggernaut | Quick flash |
| **Teleport** | In/out | Special transfer modes |
| **Destruction** | Lamp breaking | Triggered by damage |
| **Sound-only** | Metallic clang, fist hit | No visual |

---

## 26.10 Effect API

```c
// Create new effect at position
short new_effect(world_point3d *origin, short polygon_index,
                 short type, angle facing);

// Update all effects (called each tick)
void update_effects(void);

// Remove specific effect
void remove_effect(short effect_index);

// Remove all temporary effects (level change)
void remove_all_nonpersistent_effects(void);

// Load/unload effect graphics
void mark_effect_collections(short type, boolean loading);

// Teleportation helpers
void teleport_object_in(short object_index);
void teleport_object_out(short object_index);
```

---

## 26.11 Summary

Marathon's effect system provides:

- **64 effect types** covering all combat feedback
- **Slot-based management** with 64 concurrent effects
- **Delayed effects** for staggered visuals
- **Teleportation support** with twin object tracking
- **Sound integration** for audio-visual sync

### Key Source Files

| File | Purpose |
|------|---------|
| `effects.c` | Effect creation and updates |
| `effects.h` | Effect types and structures |
| `effect_definitions.h` | Effect type data |

---

*Next: [Chapter 27: Scenery Objects](27_scenery.md) - Static decorations and destructibles*
