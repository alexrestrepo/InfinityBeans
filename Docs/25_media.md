# Chapter 25: Media/Liquid System

## Water, Lava, Goo, and Environmental Effects

> **For Porting:** The media system in `media.c` and `media.h` is fully portable. Media rendering integrates with the standard texture system.

---

## 25.1 What Problem Are We Solving?

Marathon needs dynamic liquid surfaces that:

- **Rise and fall** - Water levels change over time
- **Damage players** - Lava and goo hurt on contact
- **Affect movement** - Swimming physics differ from walking
- **Provide feedback** - Visual tints and sounds when submerged
- **Flow with currents** - Push players and objects

---

## 25.2 Media Types

```c
enum /* media types */ {
    _media_water,   // Safe, swimmable
    _media_lava,    // Damaging, glowing
    _media_goo,     // Damaging, sticky
    _media_sewage,  // Safe, murky
    _media_jjaro,   // Safe, alien
    NUMBER_OF_MEDIA_TYPES  // 5
};

#define MAXIMUM_MEDIAS_PER_MAP 16
```

### Media Properties

| Type | Damage | Frequency | Screen Tint | Notes |
|------|--------|-----------|-------------|-------|
| Water | None | - | Blue | Safe, normal swimming |
| Lava | 16 pts | Every 16 ticks | Orange/red | High damage |
| Goo | 8 pts | Every 8 ticks | Green | Frequent damage |
| Sewage | None | - | Brown | Safe, murky visuals |
| Jjaro | None | - | Brown | Alien technology |

---

## 25.3 Media Data Structure

```c
struct media_data {  // 32 bytes
    short type;              // _media_water, _media_lava, etc.
    word flags;              // Sound obstruction flags

    /* Light controls height animation!
       height = low + (high-low) * light_intensity */
    short light_index;

    // Current/flow properties
    angle current_direction;          // Flow direction (0-511)
    world_distance current_magnitude; // Flow speed

    // Height range
    world_distance low, high;         // Min/max heights

    // Texture scrolling
    world_point2d origin;             // Texture offset

    // Current state
    world_distance height;            // Current calculated height

    // Rendering
    fixed minimum_light_intensity;
    shape_descriptor texture;
    short transfer_mode;

    short unused[2];
};
```

---

## 25.4 Height Animation via Lights

Marathon cleverly reuses the light system for media animation:

```c
#define CALCULATE_MEDIA_HEIGHT(m) \
    ((m)->low + FIXED_INTEGERAL_PART( \
        ((m)->high - (m)->low) * get_light_intensity((m)->light_index)))
```

### Visualization

```
Light Intensity:  0.0                              1.0
                   │                                │
                   ▼                                ▼
Media Height:     low                             high
                   │                                │
                   ▼                                ▼

Intensity = 0.0:           Intensity = 0.5:           Intensity = 1.0:
┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
│                 │       │                 │       │░░░░░░░░░░░░░░░░░│
│                 │       │                 │       │░░░ FLOODED ░░░░│
│                 │       │░░░░░░░░░░░░░░░░░│       │░░░░░░░░░░░░░░░░░│
│░░░░ water ░░░░░│       │░░░ water ░░░░░░│       │░░░░░░░░░░░░░░░░░│
└─────────────────┘       └─────────────────┘       └─────────────────┘
    (at low)                 (at midpoint)              (at high)

Benefits:
• Reuse light animation system (flicker, fade, strobe)
• Smooth transitions
• Rising water tied to light switches
• No additional code needed
```

---

## 25.5 Media Sounds

```c
enum /* media sounds */ {
    _media_snd_feet_entering,     // Splash when walking in
    _media_snd_feet_leaving,      // Exit splash
    _media_snd_head_entering,     // Submerge sound
    _media_snd_head_leaving,      // Surface sound
    _media_snd_splashing,         // Walking through
    _media_snd_ambient_over,      // Ambient above surface
    _media_snd_ambient_under,     // Ambient below surface
    _media_snd_platform_entering, // Platform into liquid
    _media_snd_platform_leaving,  // Platform out of liquid
    NUMBER_OF_MEDIA_SOUNDS        // 9
};
```

### Sound Transition Flow

```
Player Movement Through Media:

    ABOVE SURFACE          ENTERING             SUBMERGED
    ─────────────         ──────────           ───────────
    ambient_over    →    feet_entering   →    ambient_under
                              │                     │
                              ▼                     ▼
                         splashing ────────── splashing
                              │
                    head_entering ──────────────────┘
                              │
    ambient_over   ←    head_leaving    ←    ambient_under
         ↑                                          │
         └────────── feet_leaving ◄─────────────────┘
```

---

## 25.6 Media Detonation Effects

When projectiles hit liquid surfaces:

```c
enum /* detonation sizes */ {
    _small_media_detonation_effect,    // Bullet impacts
    _medium_media_detonation_effect,   // Grenades
    _large_media_detonation_effect,    // Rockets
    _large_media_emergence_effect,     // Object surfacing
    NUMBER_OF_MEDIA_DETONATION_TYPES
};
```

### Splash Effects by Media Type

| Media | Small | Medium | Large | Emergence |
|-------|-------|--------|-------|-----------|
| Water | `_effect_small_water_splash` | `_effect_medium_water_splash` | `_effect_large_water_splash` | `_effect_large_water_emergence` |
| Lava | `_effect_small_lava_splash` | `_effect_medium_lava_splash` | `_effect_large_lava_splash` | `_effect_large_lava_emergence` |
| Goo | `_effect_small_goo_splash` | `_effect_medium_goo_splash` | `_effect_large_goo_splash` | `_effect_large_goo_emergence` |
| Sewage | `_effect_small_sewage_splash` | `_effect_medium_sewage_splash` | `_effect_large_sewage_splash` | `_effect_large_sewage_emergence` |
| Jjaro | `_effect_small_jjaro_splash` | `_effect_medium_jjaro_splash` | `_effect_large_jjaro_splash` | `_effect_large_jjaro_emergence` |

---

## 25.7 Media Flow (Currents)

```c
void update_media_origin(struct media_data *media) {
    // Scroll texture based on current direction and magnitude
    media->origin.x = WORLD_FRACTIONAL_PART(media->origin.x +
        ((cosine_table[media->current_direction] * media->current_magnitude) >> TRIG_SHIFT));
    media->origin.y = WORLD_FRACTIONAL_PART(media->origin.y +
        ((sine_table[media->current_direction] * media->current_magnitude) >> TRIG_SHIFT));
}
```

### Current Effect on Player

```
Player in Flowing Media:

    current_direction = 128 (north)
    current_magnitude = 100

    ┌─────────────────────────┐
    │  ░░░░░░░░░░░░░░░░░░░░░  │
    │  ░░░░░░░░░░░░░░░░░░░░░  │     Flow Direction
    │  ░░░░░ ▲ ░░░░░░░░░░░░░  │           ↑
    │  ░░░░░│░░░░░░░░░░░░░░░  │           │
    │  ░░░░░●░░░░░░░░░░░░░░░  │           N
    │  ░░░░ Player ░░░░░░░░░  │
    └─────────────────────────┘

Each tick:
    external_velocity += current_magnitude / 32
    direction = current_direction

Player pushed north at rate of ~3 units/tick
```

---

## 25.8 Submerged Effects

```c
short get_media_submerged_fade_effect(short media_index) {
    struct media_data *media = get_media_data(media_index);
    return media_definitions[media->type].submerged_fade_effect;
}
```

### Underwater Visual Changes

```
┌─────────────────────────────────────────────────────────────┐
│                    ABOVE SURFACE                             │
│   Normal rendering, full visibility, normal sounds           │
├─────────────────────────────────────────────────────────────┤
│ ～～～～～～～～ SURFACE LINE ～～～～～～～～～～～～～～～  │
├─────────────────────────────────────────────────────────────┤
│                    BELOW SURFACE                             │
│   • Screen fade applied (blue/green/orange tint)             │
│   • Ambient sound changes to underwater                      │
│   • Oxygen starts depleting (if applicable)                  │
│   • Movement slowed (gravity/terminal velocity halved)       │
│   • Can swim up with jump/swim key                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 25.9 Media Damage

```c
struct damage_definition *get_media_damage(short media_index, fixed scale) {
    struct media_data *media = get_media_data(media_index);
    struct media_definition *definition = &media_definitions[media->type];

    // Check damage frequency mask
    if (dynamic_world->tick_count & definition->damage_frequency)
        return NULL;  // Not this tick

    return &definition->damage;
}
```

### Damage Frequency Masks

| Media | Mask | Frequency | Damage/Hit |
|-------|------|-----------|------------|
| Lava | `0x0F` | Every 16 ticks (~0.5 sec) | 16 points |
| Goo | `0x07` | Every 8 ticks (~0.25 sec) | 8 points |
| Water | `0x00` | Never | 0 |
| Sewage | `0x00` | Never | 0 |
| Jjaro | `0x00` | Never | 0 |

### Tick Mask Visualization

```
Lava damage (mask 0x0F = binary 00001111):

Tick:    0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 ...
tick&0xF: 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15  0  1 ...
Damage:  ★  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  ★  - ...

★ = damage applied (when tick & mask == 0)
- = no damage

Result: Damage every 16 ticks
```

---

## 25.10 Media API

```c
// Create new media instance
short new_media(struct media_data *data);

// Update all media each tick
void update_medias(void);

// Get detonation effect for projectile impact
void get_media_detonation_effect(short media_index, short type, short *detonation_effect);

// Get sound for media event
short get_media_sound(short media_index, short type);

// Get screen tint when submerged
short get_media_submerged_fade_effect(short media_index);

// Get damage definition (NULL if no damage this tick)
struct damage_definition *get_media_damage(short media_index, fixed scale);

// Check if media type can exist in environment
boolean media_in_environment(short media_type, short environment_code);
```

---

## 25.11 Summary

Marathon's media system provides:

- **5 liquid types** with distinct behaviors
- **Light-driven height** animation
- **9 sound events** for immersive audio
- **Current flow** affecting player movement
- **Damage frequency** via tick masking
- **Visual tints** when submerged

### Key Source Files

| File | Purpose |
|------|---------|
| `media.c` | Media update and queries |
| `media.h` | Structures and constants |
| `physics.c` | Swimming/underwater movement |
| `fades.c` | Submerged screen tints |

---

*Next: [Chapter 26: Visual Effects System](26_effects.md) - Explosions, splashes, and particles*
