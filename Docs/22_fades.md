# Chapter 22: Screen Effects & Fades

## Damage Flashes, Transitions, and Color Effects

> **For Porting:** The fade logic in `fades.c` is portable. Replace the color table manipulation with your graphics API's equivalent (e.g., shader uniforms, palette modification, or post-processing).

---

## 22.1 What Problem Are We Solving?

Players need immediate visual feedback for:

- **Damage taken** - Flash red when hurt
- **Powerup collected** - Flash on pickup
- **Environmental hazards** - Tint screen in lava/water
- **Cinematic transitions** - Fade in/out between scenes
- **Special effects** - Teleportation, explosions

---

## 22.2 Fade Type Enumeration

```c
enum /* fade types */ {
    // Cinematic fades (level transitions)
    _start_cinematic_fade_in,   // Force black immediately
    _cinematic_fade_in,         // Fade from black
    _long_cinematic_fade_in,    // Slow fade from black
    _cinematic_fade_out,        // Fade to black
    _end_cinematic_fade_out,    // Force black immediately

    // Damage fades (flash and decay)
    _fade_red,                  // Bullets, fist
    _fade_big_red,              // Heavy damage
    _fade_bonus,                // Item pickup
    _fade_bright,               // Teleporting
    _fade_long_bright,          // Nuclear detonation
    _fade_yellow,               // Explosions
    _fade_big_yellow,           // Big explosions
    _fade_purple,               // Rare effect
    _fade_cyan,                 // Fighter weapons
    _fade_white,                // Absorbed damage
    _fade_big_white,            // Heavy absorbed
    _fade_orange,               // Flamethrower
    _fade_long_orange,          // Lava damage
    _fade_green,                // Hunter projectile
    _fade_long_green,           // Alien goo
    _fade_static,               // Compiler projectile
    _fade_negative,             // Minor fusion
    _fade_big_negative,         // Major fusion
    _fade_flicker_negative,     // Hummer projectile
    _fade_dodge_purple,         // Alien weapon
    _fade_burn_cyan,            // Armageddon electricity
    _fade_dodge_yellow,         // Armageddon projectile
    _fade_burn_green,           // Hunter projectile burn

    // Environmental tints (persistent while submerged)
    _fade_tint_green,           // Under goo
    _fade_tint_blue,            // Under water
    _fade_tint_orange,          // Under lava
    _fade_tint_gross,           // Under sewage

    NUMBER_OF_FADE_TYPES
};
```

---

## 22.3 Fade Categories

### Cinematic Fades

Used for level transitions and cutscenes:

```
_cinematic_fade_in:

Frame:  0     5    10    15    20    25    30
        │     │     │     │     │     │     │
Black ▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ Normal
      └─────────────────────────────────────┘
              Gradual brightness increase
```

### Damage Fades

Flash immediately, then decay:

```
_fade_red (bullet hit):

Frame:  0     5    10    15    20
        │     │     │     │     │
     ████████████░░░░░░░░░░░░░░░░░░ Normal
     └─── Flash ──┘└── Decay ────┘

Peak intensity at frame 0, linear decay to normal
```

### Environmental Tints

Persistent color overlay while condition exists:

```
_fade_tint_blue (underwater):

Above water:  Normal colors
              │
Enter water:  Blue tint applied ────────────────
              │                                 │
Exit water:   Normal colors ◄───────────────────┘

Tint persists until player surfaces
```

---

## 22.4 Fade Effect Types

```c
enum /* effect types */ {
    _effect_under_water,    // Blue tint
    _effect_under_lava,     // Orange tint
    _effect_under_sewage,   // Brown tint
    _effect_under_goo,      // Green tint
    NUMBER_OF_FADE_EFFECT_TYPES
};
```

### Effect-to-Fade Mapping

| Environment | Effect Type | Fade Type |
|-------------|-------------|-----------|
| Water | `_effect_under_water` | `_fade_tint_blue` |
| Lava | `_effect_under_lava` | `_fade_tint_orange` |
| Sewage | `_effect_under_sewage` | `_fade_tint_gross` |
| Goo | `_effect_under_goo` | `_fade_tint_green` |

---

## 22.5 Fade API

```c
// Start a fade effect
void start_fade(short type);

// Stop current fade immediately
void stop_fade(void);

// Check if fade is still active
boolean fade_finished(void);

// Get duration of fade type in ticks
short get_fade_period(short type);

// Set persistent environmental effect
void set_fade_effect(short type);

// Update fades each tick (returns TRUE if screen changed)
boolean update_fades(void);
```

### Explicit Fade Control

```c
// For custom color table manipulation
void explicit_start_fade(
    short type,
    struct color_table *original_color_table,
    struct color_table *animated_color_table
);

// Immediate full fade (no animation)
void full_fade(
    short type,
    struct color_table *original_color_table
);
```

---

## 22.6 Gamma Correction

```c
enum {
    NUMBER_OF_GAMMA_LEVELS = 8,
    DEFAULT_GAMMA_LEVEL = 2
};

void gamma_correct_color_table(
    struct color_table *uncorrected_color_table,
    struct color_table *corrected_color_table,
    short gamma_level
);
```

### Gamma Curve Visualization

```
Gamma Levels (0-7):

Output     Level 0 (darkest)    Level 7 (brightest)
Brightness        │                    │
    255 ┤         │    ╱               │   ─────────
        │         │   ╱                │  ╱
        │         │  ╱                 │ ╱
    128 ┤         │ ╱                  │╱
        │         │╱                   ╱
        │        ╱│                  ╱ │
      0 ┼───────╱─┼────────────────╱───┼────────
        0       128       255    0       128       255
                Input                  Input

Higher gamma = brighter midtones
```

---

## 22.7 Color Table Animation

Fades work by interpolating between color tables:

```
Original Color Table          Animated Color Table
┌───────────────────┐         ┌───────────────────┐
│ R: 255            │   ──►   │ R: 255 + tint     │
│ G: 200            │   ──►   │ G: 200 + tint     │
│ B: 150            │   ──►   │ B: 150 + tint     │
│ ...               │         │ ...               │
│ (256 entries)     │         │ (256 entries)     │
└───────────────────┘         └───────────────────┘

Fade calculation per entry:
  animated[i] = original[i] + ((target[i] - original[i]) * phase / period)
```

---

## 22.8 Damage Fade Trigger Flow

```
Player takes damage:
    │
    ├─► Calculate damage type
    │     └─► Explosion: _fade_yellow
    │     └─► Bullet: _fade_red
    │     └─► Fusion: _fade_negative
    │
    ├─► Calculate intensity
    │     └─► damage > threshold ? _fade_big_* : _fade_*
    │
    └─► start_fade(fade_type)
          │
          ├─► Frame 0: Peak color shift
          ├─► Frame 1-N: Linear decay
          └─► Frame N: Normal restored
```

---

## 22.9 Implementation Details

### Fade State

```c
static struct {
    short type;                    // Current fade type
    short phase;                   // Current frame in fade
    short period;                  // Total fade duration
    struct color_table original;   // Colors before fade
    struct color_table target;     // Target colors
    boolean active;                // Fade in progress?
} fade_state;
```

### Update Loop

```c
boolean update_fades(void) {
    if (!fade_state.active) return FALSE;

    fade_state.phase++;

    if (fade_state.phase >= fade_state.period) {
        // Fade complete
        fade_state.active = FALSE;
        restore_original_colors();
        return TRUE;
    }

    // Interpolate colors
    for (int i = 0; i < 256; i++) {
        float t = (float)fade_state.phase / fade_state.period;
        animated_table[i] = lerp(fade_state.original[i],
                                  fade_state.target[i], t);
    }

    apply_color_table(&animated_table);
    return TRUE;
}
```

---

## 22.10 Summary

Marathon's fade system provides:

- **30 fade types** for varied visual feedback
- **Cinematic fades** for level transitions
- **Damage flashes** with intensity scaling
- **Environmental tints** for submerged states
- **Gamma correction** for brightness adjustment

### Key Source Files

| File | Purpose |
|------|---------|
| `fades.c` | Fade logic and color math |
| `fades.h` | Fade type definitions |
| `screen.c` | Color table application |

---

*Next: [Chapter 23: View Bobbing & Camera](23_camera.md) - Player view and weapon sway*
