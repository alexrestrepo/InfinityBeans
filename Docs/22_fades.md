# Chapter 22: Screen Effects & Fades

## Damage Flashes, Transitions, and Color Effects

> **Source files**: `fades.c`, `fades.h`, `screen.c`
> **Related chapters**: [Chapter 16: Damage](16_damage.md), [Chapter 25: Media](25_media.md)

> **For Porting:** The fade logic in `fades.c` is portable (568 lines, no Mac dependencies). Replace the color table manipulation with your graphics API's equivalent (e.g., shader uniforms, palette modification, or post-processing).

---

## 22.1 What Problem Are We Solving?

Players need immediate visual feedback for:

- **Damage taken** - Flash red when hurt
- **Powerup collected** - Flash on pickup
- **Environmental hazards** - Tint screen in lava/water
- **Cinematic transitions** - Fade in/out between scenes
- **Special effects** - Teleportation, explosions

**The constraints:**
- Must work with indexed color (palette-based rendering)
- Must animate smoothly over multiple frames
- Must support priority system (important fades override weaker ones)

---

## 22.2 Gamma Level Constants (`fades.h:8-12`)

```c
enum
{
    NUMBER_OF_GAMMA_LEVELS= 8,
    DEFAULT_GAMMA_LEVEL= 2
};
```

---

## 22.3 Fade Type Enumeration (`fades.h:14-52`)

```c
enum /* fade types */
{
    _start_cinematic_fade_in, /* 0 - force all colors to black immediately */
    _cinematic_fade_in,       /* 1 - fade in from black */
    _long_cinematic_fade_in,  /* 2 - slow fade in from black */
    _cinematic_fade_out,      /* 3 - fade out from black */
    _end_cinematic_fade_out,  /* 4 - force all colors from black immediately */

    _fade_red,                /* 5 - bullets and fist */
    _fade_big_red,            /* 6 - bigger bullets and fists */
    _fade_bonus,              /* 7 - picking up items */
    _fade_bright,             /* 8 - teleporting */
    _fade_long_bright,        /* 9 - nuclear monster detonations */
    _fade_yellow,             /* 10 - explosions */
    _fade_big_yellow,         /* 11 - big explosions */
    _fade_purple,             /* 12 - ? */
    _fade_cyan,               /* 13 - fighter staves and projectiles */
    _fade_white,              /* 14 - absorbed */
    _fade_big_white,          /* 15 - rocket (probably) absorbed */
    _fade_orange,             /* 16 - flamethrower */
    _fade_long_orange,        /* 17 - marathon lava */
    _fade_green,              /* 18 - hunter projectile */
    _fade_long_green,         /* 19 - alien green goo */
    _fade_static,             /* 20 - compiler projectile */
    _fade_negative,           /* 21 - minor fusion projectile */
    _fade_big_negative,       /* 22 - major fusion projectile */
    _fade_flicker_negative,   /* 23 - hummer projectile */
    _fade_dodge_purple,       /* 24 - alien weapon */
    _fade_burn_cyan,          /* 25 - armageddon beast electricity */
    _fade_dodge_yellow,       /* 26 - armageddon beast projectile */
    _fade_burn_green,         /* 27 - hunter projectile */

    _fade_tint_green,         /* 28 - under goo */
    _fade_tint_blue,          /* 29 - under water */
    _fade_tint_orange,        /* 30 - under lava */
    _fade_tint_gross,         /* 31 - under sewage */

    NUMBER_OF_FADE_TYPES
};
```

---

## 22.4 Fade Effect Types (`fades.h:54-61`)

```c
enum /* effect types */
{
    _effect_under_water,    /* 0 */
    _effect_under_lava,     /* 1 */
    _effect_under_sewage,   /* 2 */
    _effect_under_goo,      /* 3 */
    NUMBER_OF_FADE_EFFECT_TYPES
};
```

### Effect-to-Fade Mapping (`fades.c:150-156`)

```c
struct fade_effect_definition fade_effect_definitions[NUMBER_OF_FADE_EFFECT_TYPES]=
{
    {_fade_tint_blue, 3*FIXED_ONE/4},   /* _effect_under_water */
    {_fade_tint_orange, 3*FIXED_ONE/4}, /* _effect_under_lava */
    {_fade_tint_gross, 3*FIXED_ONE/4},  /* _effect_under_sewage */
    {_fade_tint_green, 3*FIXED_ONE/4},  /* _effect_under_goo */
};
```

| Environment | Effect Type | Fade Type | Transparency |
|-------------|-------------|-----------|--------------|
| Water | `_effect_under_water` | `_fade_tint_blue` | 75% |
| Lava | `_effect_under_lava` | `_fade_tint_orange` | 75% |
| Sewage | `_effect_under_sewage` | `_fade_tint_gross` | 75% |
| Goo | `_effect_under_goo` | `_fade_tint_green` | 75% |

---

## 22.5 Fade Categories

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

## 22.6 Fade Timing Constants (`fades.c:36-42`)

```c
enum
{
    ADJUSTED_TRANSPARENCY_DOWNSHIFT= 8,

    MINIMUM_FADE_RESTART_TICKS= MACHINE_TICKS_PER_SECOND/2,  /* 30 ticks */
    MINIMUM_FADE_UPDATE_TICKS= MACHINE_TICKS_PER_SECOND/8    /* ~7 ticks */
};
```

---

## 22.7 Fade Definition Structure (`fades.c:67-77`)

```c
struct fade_definition
{
    fade_proc proc;                              /* Color transformation function */
    struct rgb_color color;                      /* Target/blend color */
    fixed initial_transparency, final_transparency; /* in [0,FIXED_ONE] */

    short period;                                /* Duration in ticks */

    word flags;
    short priority;                              /* Higher is higher */
};
```

### Fade Flags (`fades.c:61-65`)

```c
enum
{
    _full_screen_flag= 0x0001,        /* Affects entire screen, not just world */
    _random_transparency_flag= 0x0002 /* Add randomness to transparency */
};
```

---

## 22.8 Fade Data Structure (`fades.c:82-94`)

```c
struct fade_data
{
    word flags; /* [active.1] [unused.15] */

    short type;
    short fade_effect_type;

    long start_tick;
    long last_update_tick;

    struct color_table *original_color_table;
    struct color_table *animated_color_table;
};
```

### Active Status Macros (`fades.c:79-80`)

```c
#define FADE_IS_ACTIVE(f) ((f)->flags&(word)0x8000)
#define SET_FADE_ACTIVE_STATUS(f,s) ((s)?((f)->flags|=(word)0x8000):((f)->flags&=(word)~0x8000))
```

---

## 22.9 Fade Definitions Array (`fades.c:111-148`)

Each fade type is configured with color, duration, and processing function:

```c
static struct fade_definition fade_definitions[NUMBER_OF_FADE_TYPES]=
{
    /* Cinematic fades */
    {tint_color_table, {0, 0, 0}, FIXED_ONE, FIXED_ONE, 0, _full_screen_flag, 0}, /* _start_cinematic_fade_in */
    {tint_color_table, {0, 0, 0}, FIXED_ONE, 0, MACHINE_TICKS_PER_SECOND/2, _full_screen_flag, 0}, /* _cinematic_fade_in */
    {tint_color_table, {0, 0, 0}, FIXED_ONE, 0, 3*MACHINE_TICKS_PER_SECOND/2, _full_screen_flag, 0}, /* _long_cinematic_fade_in */
    {tint_color_table, {0, 0, 0}, 0, FIXED_ONE, MACHINE_TICKS_PER_SECOND/2, _full_screen_flag, 0}, /* _cinematic_fade_out */
    {tint_color_table, {0, 0, 0}, 0, 0, 0, _full_screen_flag, 0}, /* _end_cinematic_fade_out */

    /* Damage fades */
    {tint_color_table, {65535, 0, 0}, (3*FIXED_ONE)/4, 0, MACHINE_TICKS_PER_SECOND/4, 0, 0}, /* _fade_red */
    {tint_color_table, {65535, 0, 0}, FIXED_ONE, 0, (3*MACHINE_TICKS_PER_SECOND)/4, 0, 25}, /* _fade_big_red */
    {tint_color_table, {0, 65535, 0}, FIXED_ONE_HALF, 0, MACHINE_TICKS_PER_SECOND/4, 0, 0}, /* _fade_bonus */
    {tint_color_table, {65535, 65535, 50000}, FIXED_ONE, 0, MACHINE_TICKS_PER_SECOND/3, 0, 0}, /* _fade_bright */
    {tint_color_table, {65535, 65535, 50000}, FIXED_ONE, 0, 4*MACHINE_TICKS_PER_SECOND, 0, 100}, /* _fade_long_bright */
    {tint_color_table, {65535, 65535, 0}, FIXED_ONE, 0, MACHINE_TICKS_PER_SECOND/2, 0, 50}, /* _fade_yellow */
    {tint_color_table, {65535, 65535, 0}, FIXED_ONE, 0, MACHINE_TICKS_PER_SECOND, 0, 75}, /* _fade_big_yellow */
    /* ... more fade definitions ... */

    /* Special effects */
    {randomize_color_table, {0, 0, 0}, FIXED_ONE, 0, (3*MACHINE_TICKS_PER_SECOND)/8, 0, 0}, /* _fade_static */
    {negate_color_table, {65535, 65535, 65535}, FIXED_ONE, 0, MACHINE_TICKS_PER_SECOND/2, 0, 0}, /* _fade_negative */
    {negate_color_table, {65535, 65535, 65535}, FIXED_ONE, 0, (3*MACHINE_TICKS_PER_SECOND)/2, 0, 25}, /* _fade_big_negative */

    /* Environmental tints */
    {soft_tint_color_table, {137*256, 0, 137*256}, FIXED_ONE, 0, 2*MACHINE_TICKS_PER_SECOND, 0, 0}, /* _fade_tint_green */
    {soft_tint_color_table, {0, 0, 65535}, FIXED_ONE, 0, 2*MACHINE_TICKS_PER_SECOND, 0, 0}, /* _fade_tint_blue */
    {soft_tint_color_table, {65535, 16384, 0}, FIXED_ONE, 0, 2*MACHINE_TICKS_PER_SECOND, 0, 0}, /* _fade_tint_orange */
    {soft_tint_color_table, {32768, 65535, 0}, FIXED_ONE, 0, 2*MACHINE_TICKS_PER_SECOND, 0, 0}, /* _fade_tint_gross */
};
```

---

## 22.10 Fade API (`fades.h:63-79`)

```c
void initialize_fades(void);
boolean update_fades(void);

void start_fade(short type);
void stop_fade(void);
boolean fade_finished(void);

void set_fade_effect(short type);

short get_fade_period(short type);

void gamma_correct_color_table(struct color_table *uncorrected_color_table,
    struct color_table *corrected_color_table, short gamma_level);

void explicit_start_fade(short type, struct color_table *original_color_table,
    struct color_table *animated_color_table);
void full_fade(short type, struct color_table *original_color_table);
```

---

## 22.11 Fade Update Function (`fades.c:193-226`)

```c
boolean update_fades(
    void)
{
    if (FADE_IS_ACTIVE(fade))
    {
        struct fade_definition *definition= get_fade_definition(fade->type);
        long tick_count= machine_tick_count();
        boolean update= FALSE;
        fixed transparency;
        short phase;

        if ((phase= tick_count-fade->start_tick)>=definition->period)
        {
            transparency= definition->final_transparency;
            SET_FADE_ACTIVE_STATUS(fade, FALSE);

            update= TRUE;
        }
        else
        {
            if (tick_count-fade->last_update_tick>=MINIMUM_FADE_UPDATE_TICKS)
            {
                transparency= definition->initial_transparency +
                    (phase*(definition->final_transparency-definition->initial_transparency))/definition->period;
                if (definition->flags&_random_transparency_flag)
                    transparency+= FADES_RANDOM()%(definition->final_transparency-transparency);

                update= TRUE;
            }
        }

        if (update) recalculate_and_display_color_table(fade->type, transparency,
            fade->original_color_table, fade->animated_color_table);
    }

    return FADE_IS_ACTIVE(fade) ? TRUE : FALSE;
}
```

---

## 22.12 Gamma Correction (`fades.c:158-168`, `fades.c:340-362`)

### Gamma Values (`fades.c:158-168`)

```c
static float actual_gamma_values[NUMBER_OF_GAMMA_LEVELS]=
{
    1.3,   /* 0 - darkest */
    1.15,  /* 1 */
    1.0,   /* 2 - default */
    0.95,  /* 3 */
    0.90,  /* 4 */
    0.85,  /* 5 */
    0.77,  /* 6 */
    0.70   /* 7 - brightest */
};
```

### Gamma Correction Function (`fades.c:340-362`)

```c
void gamma_correct_color_table(
    struct color_table *uncorrected_color_table,
    struct color_table *corrected_color_table,
    short gamma_level)
{
    short i;
    float gamma;
    struct rgb_color *uncorrected= uncorrected_color_table->colors;
    struct rgb_color *corrected= corrected_color_table->colors;

    assert(gamma_level>=0 && gamma_level<NUMBER_OF_GAMMA_LEVELS);
    gamma= actual_gamma_values[gamma_level];

    corrected_color_table->color_count= uncorrected_color_table->color_count;
    for (i= 0; i<uncorrected_color_table->color_count; ++i, ++corrected, ++uncorrected)
    {
        corrected->red= pow(uncorrected->red/65535.0, gamma)*65535.0;
        corrected->green= pow(uncorrected->green/65535.0, gamma)*65535.0;
        corrected->blue= pow(uncorrected->blue/65535.0, gamma)*65535.0;
    }
}
```

### Gamma Level Effects

| Level | Gamma Value | Effect | Use Case |
|-------|-------------|--------|----------|
| 0 | 1.3 | Very dark | High ambient light rooms |
| 1 | 1.15 | Dark | Bright monitors |
| 2 | 1.0 | Default | Normal conditions |
| 3-4 | 0.95-0.90 | Brighter | Dark rooms |
| 5-7 | 0.85-0.70 | Very bright | Maximum visibility |

---

## 22.13 Color Table Transformation Functions

### Tint (`fades.c:415-435`)

```c
static void tint_color_table(
    struct color_table *original_color_table,
    struct color_table *animated_color_table,
    struct rgb_color *color,
    fixed transparency)
{
    short i;
    struct rgb_color *unadjusted= original_color_table->colors;
    struct rgb_color *adjusted= animated_color_table->colors;
    short adjusted_transparency= transparency>>ADJUSTED_TRANSPARENCY_DOWNSHIFT;

    animated_color_table->color_count= original_color_table->color_count;
    for (i= 0; i<original_color_table->color_count; ++i, ++adjusted, ++unadjusted)
    {
        adjusted->red= unadjusted->red + (((color->red-unadjusted->red)*adjusted_transparency)>>(FIXED_FRACTIONAL_BITS-ADJUSTED_TRANSPARENCY_DOWNSHIFT));
        adjusted->green= unadjusted->green + (((color->green-unadjusted->green)*adjusted_transparency)>>(FIXED_FRACTIONAL_BITS-ADJUSTED_TRANSPARENCY_DOWNSHIFT));
        adjusted->blue= unadjusted->blue + (((color->blue-unadjusted->blue)*adjusted_transparency)>>(FIXED_FRACTIONAL_BITS-ADJUSTED_TRANSPARENCY_DOWNSHIFT));
    }
}
```

### Randomize (`fades.c:437-464`)

```c
static void randomize_color_table(
    struct color_table *original_color_table,
    struct color_table *animated_color_table,
    struct rgb_color *color,
    fixed transparency)
{
    /* Creates static effect by adding random noise to colors */
    word mask, adjusted_transparency= PIN(transparency, 0, 0xffff);

    /* calculate a mask with all bits up to high-bit in transparency */
    for (mask= 0;~mask & adjusted_transparency;mask= (mask<<1)|1)
        ;

    for (i= 0; i<original_color_table->color_count; ++i, ++adjusted, ++unadjusted)
    {
        adjusted->red= unadjusted->red + (FADES_RANDOM()&mask);
        adjusted->green= unadjusted->green + (FADES_RANDOM()&mask);
        adjusted->blue= unadjusted->blue + (FADES_RANDOM()&mask);
    }
}
```

### Negate (`fades.c:467-493`)

```c
static void negate_color_table(
    struct color_table *original_color_table,
    struct color_table *animated_color_table,
    struct rgb_color *color,
    fixed transparency)
{
    /* Creates inverse/negative effect - used for fusion hits */
    transparency= FIXED_ONE-transparency;
    for (i= 0; i<original_color_table->color_count; ++i, ++adjusted, ++unadjusted)
    {
        adjusted->red= (unadjusted->red>0x8000) ?
            CEILING((unadjusted->red^color->red)+transparency, (long)unadjusted->red) :
            FLOOR((unadjusted->red^color->red)-transparency, (long)unadjusted->red);
        /* ... same for green and blue ... */
    }
}
```

---

## 22.14 Available Fade Procedures

| Procedure | Effect | Used For |
|-----------|--------|----------|
| `tint_color_table` | Blend toward solid color | Most damage fades, cinematics |
| `randomize_color_table` | Static noise | Compiler projectile hit |
| `negate_color_table` | Inverse colors | Fusion bolt |
| `dodge_color_table` | Photoshop-like dodge | Alien weapons |
| `burn_color_table` | Photoshop-like burn | Electricity effects |
| `soft_tint_color_table` | Intensity-aware tint | Environmental (underwater) |

---

## 22.15 Summary

Marathon's fade system provides:

- **32 fade types** for varied visual feedback (`fades.h:14-52`)
- **Cinematic fades** for level transitions (types 0-4)
- **Damage flashes** with intensity scaling and priority (`fades.c:111-148`)
- **Environmental tints** for submerged states (types 28-31)
- **Gamma correction** with 8 levels (`fades.c:158-168`)

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `NUMBER_OF_FADE_TYPES` | 32 | `fades.h:51` |
| `NUMBER_OF_FADE_EFFECT_TYPES` | 4 | `fades.h:60` |
| `NUMBER_OF_GAMMA_LEVELS` | 8 | `fades.h:10` |
| `DEFAULT_GAMMA_LEVEL` | 2 | `fades.h:11` |
| `MINIMUM_FADE_RESTART_TICKS` | 30 | `fades.c:40` |
| `MINIMUM_FADE_UPDATE_TICKS` | ~7 | `fades.c:41` |

### Key Source Files

| File | Purpose |
|------|---------|
| `fades.c` | Fade logic and color math (568 lines) |
| `fades.h` | Fade type definitions, prototypes (80 lines) |
| `screen.c` | Color table application (`animate_screen_clut`) |

---

## 22.16 See Also

- [Chapter 16: Damage](16_damage.md) — Damage types trigger fades
- [Chapter 25: Media](25_media.md) — Media types trigger environmental tints
- [Chapter 7: Game Loop](07_game_loop.md) — `update_fades()` called each tick

---

*Next: [Chapter 23: View Bobbing & Camera](23_camera.md) - Player view and weapon sway*
