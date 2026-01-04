# Chapter 13: Sound System

## 3D Audio, Channels, and Ambient Sounds

> **Source files**: `game_sound.c`, `game_sound.h`, `sound_definitions.h`, `sound_macintosh.c`, `ambient_sound.c`
> **Related chapters**: [Chapter 7: Game Loop](07_game_loop.md), [Chapter 25: Media](25_media.md)

> **For Porting:** Replace `sound_macintosh.c` entirely with your audio backend (recommend miniaudio.h). Keep `game_sound.c` logic (3D positioning, channel management) and redirect its output calls to your platform layer. Sound data in Sounds files is standard PCM—just need to parse the headers.

---

## 13.1 What Problem Are We Solving?

Marathon needs immersive audio that helps players locate threats and enhances the game atmosphere:

- **3D positioning** - Sounds come from specific locations in the world
- **Stereo panning** - Left/right placement based on direction
- **Distance attenuation** - Far sounds are quieter
- **Obstruction** - Walls muffle sounds
- **Ambient loops** - Environmental atmosphere
- **Channel management** - Limited channels, priority system

**The constraints:**
- Only 4-6 simultaneous sound channels on 1995 hardware
- Must convey spatial information accurately
- Must not overload the CPU with audio processing

**Marathon's solution: Prioritized Channel System**

Marathon uses a fixed number of sound channels with a priority system based on volume and distance. Sounds that would be quiet are skipped in favor of louder, more important sounds.

---

## 13.2 Understanding the Channel System

Before diving into implementation, let's understand how Marathon manages limited audio resources.

### Channel Constants (`game_sound.c:97-111`)

```c
enum
{
    MAXIMUM_OUTPUT_SOUND_VOLUME= 2*MAXIMUM_SOUND_VOLUME,
    SOUND_VOLUME_DELTA= MAXIMUM_OUTPUT_SOUND_VOLUME/NUMBER_OF_SOUND_VOLUME_LEVELS,
    DEFAULT_SOUND_LEVEL= NUMBER_OF_SOUND_VOLUME_LEVELS/3,

    ABORT_AMPLITUDE_THRESHHOLD= (MAXIMUM_SOUND_VOLUME/6),  // ~42
    MINIMUM_RESTART_TICKS= MACHINE_TICKS_PER_SECOND/12,    // ~5 ticks

    MAXIMUM_SOUND_CHANNELS= 4,
    MAXIMUM_AMBIENT_SOUND_CHANNELS= 2,

    MINIMUM_SOUND_PITCH= 1,
    MAXIMUM_SOUND_PITCH= 256*FIXED_ONE
};
```

### Volume Constants (`game_sound.h:8-14`)

```c
enum
{
    NUMBER_OF_SOUND_VOLUME_LEVELS= 8,

    MAXIMUM_SOUND_VOLUME_BITS= 8,
    MAXIMUM_SOUND_VOLUME= 1<<MAXIMUM_SOUND_VOLUME_BITS  // 256
};
```

### Channel Layout

```
Audio Channel Layout:

┌─────────────────────────────────────────────────────────────┐
│  Channel 0   │  Channel 1   │  Channel 2   │  Channel 3    │
│  (Normal)    │  (Normal)    │  (Normal)    │  (Normal)     │
├─────────────────────────────────────────────────────────────┤
│       Channel 4 (Ambient)   │       Channel 5 (Ambient)    │
└─────────────────────────────────────────────────────────────┘

Normal channels: Sound effects (weapons, monsters, items)
Ambient channels: Environmental loops (water, wind, machinery)
```

### Priority System

```
Priority Rules:
  1. Higher volume sounds preempt lower volume
  2. Closer sounds preempt distant sounds
  3. Ambient channels never preempt normal channels
  4. ABORT_AMPLITUDE_THRESHHOLD prevents quiet sounds from interrupting
```

---

## 13.3 Initialization Flags (`game_sound.h:24-33`)

```c
enum // initialization flags
{
    _stereo_flag= 0x0001,           /* play sounds in stereo */
    _dynamic_tracking_flag= 0x0002, /* tracks sound sources during idle_proc */
    _doppler_shift_flag= 0x0004,    /* adjusts sound pitch during idle_proc */
    _ambient_sound_flag= 0x0008,    /* plays and tracks ambient sounds */
    _16bit_sound_flag= 0x0010,      /* loads 16bit audio instead of 8bit */
    _more_sounds_flag= 0x0020,      /* loads all permutations; only loads #0 if false */
    _extra_memory_flag= 0x0040      /* double usual memory */
};
```

---

## 13.4 Channel Data Structure (`game_sound.c:143-164`)

```c
struct channel_data
{
    word flags;

    short sound_index;              /* sound_index being played in this channel */

    short identifier;               /* unique sound identifier (object_index) */
    struct sound_variables variables; /* volume, pitch, etc. */
    world_location3d *dynamic_source; /* can be NULL for immobile sounds */
    world_location3d source;          /* must be valid */

    unsigned long start_tick;

#ifdef mac
    SndChannelPtr channel;
    short callback_count;
#endif
};
```

### Sound Variables (`game_sound.c:130-141`)

```c
struct sound_variables
{
    fixed original_pitch, pitch;
    short left_volume, right_volume;
    short volume;
    short priority;
};
```

### Channel Flags (`game_sound.c:113-116`)

```c
enum /* channel flags */
{
    _sound_is_local= 0x0001  // .source is invalid (plays at listener)
};
```

---

## 13.5 Sound Manager Parameters (`game_sound.h:372-382`)

```c
struct sound_manager_parameters
{
    short channel_count;  /* >=0 */
    short volume;         /* [0,NUMBER_OF_SOUND_VOLUME_LEVELS) */
    word flags;           /* stereo, dynamic_tracking, etc. */

    long unused_long;
    fixed pitch;

    short unused[9];
};
```

---

## 13.6 3D Sound Positioning

### Obstruction Flags (`game_sound.h:35-40`)

```c
enum // _sound_obstructed_proc() flags
{
    _sound_was_obstructed= 0x0001,       // no clear path between source and listener
    _sound_was_media_obstructed= 0x0002, // source and listener on different sides of media
    _sound_was_media_muffled= 0x0004     // source and listener both under the same media
};
```

### Frequency Constants (`game_sound.h:42-47`)

```c
enum // frequencies
{
    _lower_frequency= FIXED_ONE-FIXED_ONE/8,   // 0.875
    _normal_frequency= FIXED_ONE,              // 1.0
    _higher_frequency= FIXED_ONE+FIXED_ONE/8   // 1.125
};
```

### Distance Attenuation

```c
// Volume decreases with distance
volume = base_volume * (MAXIMUM_SOUND_DISTANCE - distance) / MAXIMUM_SOUND_DISTANCE;
volume = MAX(volume, 0);  // Clamp to zero
```

### Stereo Panning

```c
// Calculate angle from listener to source
angle = arctangent(source.x - listener.x, source.y - listener.y);
relative_angle = NORMALIZE_ANGLE(angle - listener.facing);

// Pan based on relative angle
// 0° = center, 90° = full right, 270° = full left
left_volume = volume * (HALF_CIRCLE - ABS(relative_angle - QUARTER_CIRCLE)) / HALF_CIRCLE;
right_volume = volume * (HALF_CIRCLE - ABS(relative_angle - THREE_QUARTER_CIRCLE)) / HALF_CIRCLE;
```

### Visualization

```
                    Source
                       ●
                      /
                     / distance
                    /
                   /
        @─────────+  Listener (facing →)
                  │
                  │ angle = 45°
                  ▼

        Result: Left=70%, Right=100%
```

---

## 13.7 Sound Obstruction

Marathon checks for walls and liquid surfaces between listener and source.

### Obstruction Algorithm

```
Line-of-Sound Check:
    1. Cast ray from listener to sound source
    2. For each polygon boundary crossed:
       - If solid wall: _sound_was_obstructed
       - If media surface: _sound_was_media_obstructed
    3. If listener and source in different media:
       - _sound_was_media_muffled

Obstructed sounds:
  - Volume reduced by ~50%
  - High frequencies filtered (muffled)
```

---

## 13.8 Sound Definition Structure (`sound_definitions.h:82-105`)

```c
struct sound_definition /* 64 bytes */
{
    short sound_code;

    short behavior_index;
    word flags;

    word chance;  // play sound if AbsRandom()>=chance

    /* if low_pitch==0, use FIXED_ONE; if high_pitch==0 use low pitch */
    fixed low_pitch, high_pitch;

    /* filled in later */
    short permutations;
    word permutations_played;
    long group_offset, single_length, total_length;
    long sound_offsets[MAXIMUM_PERMUTATIONS_PER_SOUND];  // zero-based from group offset

    unsigned long last_played;  // machine ticks

    long hndl;  // zero if not loaded

    short unused[2];
};
```

### Sound Behaviors (`sound_definitions.h:18-24`)

```c
enum /* sound behaviors */
{
    _sound_is_quiet,
    _sound_is_normal,
    _sound_is_loud,
    NUMBER_OF_SOUND_BEHAVIOR_DEFINITIONS
};
```

### Sound Flags (`sound_definitions.h:26-35`)

```c
enum /* flags */
{
    _sound_cannot_be_restarted= 0x0001,
    _sound_does_not_self_abort= 0x0002,
    _sound_resists_pitch_changes= 0x0004,   // 0.5 external pitch changes
    _sound_cannot_change_pitch= 0x0008,     // no external pitch changes
    _sound_cannot_be_obstructed= 0x0010,    // ignore obstructions
    _sound_cannot_be_media_obstructed= 0x0020, // ignore media obstructions
    _sound_is_ambient= 0x0040               // only loaded with _ambient_sound_flag
};
```

### Sound Chances (`sound_definitions.h:37-49`)

```c
enum /* sound chances */
{
    _ten_percent= 32768*9/10,
    _twenty_percent= 32768*8/10,
    _thirty_percent= 32768*7/10,
    _fourty_percent= 32768*6/10,
    _fifty_percent= 32768*5/10,
    _sixty_percent= 32768*4/10,
    _seventy_percent= 32768*3/10,
    _eighty_percent= 32768*2/10,
    _ninty_percent= 32768*1/10,
    _always= 0
};
```

### Permutations (`sound_definitions.h:13-16`)

```c
enum
{
    MAXIMUM_PERMUTATIONS_PER_SOUND= 5
};
```

---

## 13.9 Sound Behavior Definitions (`sound_definitions.h:107-139`)

Marathon uses depth curves to control volume falloff based on distance:

```c
struct depth_curve_definition
{
    short maximum_volume, maximum_volume_distance;
    short minimum_volume, minimum_volume_distance;
};

struct sound_behavior_definition
{
    struct depth_curve_definition obstructed_curve, unobstructed_curve;
};

static struct sound_behavior_definition sound_behavior_definitions[]=
{
    /* _sound_is_quiet */
    {
        {0, 0, 0, 0},  /* obstructed quiet sounds make no sound */
        {MAXIMUM_SOUND_VOLUME, 0, 0, 5*WORLD_ONE},
    },

    /* _sound_is_normal */
    {
        {MAXIMUM_SOUND_VOLUME/2, 0, 0, 7*WORLD_ONE},
        {MAXIMUM_SOUND_VOLUME, WORLD_ONE, 0, 10*WORLD_ONE},
    },

    /* _sound_is_loud */
    {
        {(3*MAXIMUM_SOUND_VOLUME)/4, 0, 0, 10*WORLD_ONE},
        {MAXIMUM_SOUND_VOLUME, 2*WORLD_ONE, MAXIMUM_SOUND_VOLUME/8, 15*WORLD_ONE},
    }
};
```

---

## 13.10 Ambient Sound System

Ambient sounds provide environmental atmosphere based on the player's location.

### Ambient Sound Codes (`game_sound.h:51-83`)

```c
enum /* ambient sound codes */
{
    _ambient_snd_water,               // 0
    _ambient_snd_sewage,              // 1
    _ambient_snd_lava,                // 2
    _ambient_snd_goo,                 // 3
    _ambient_snd_under_media,         // 4
    _ambient_snd_wind,                // 5
    _ambient_snd_waterfall,           // 6
    _ambient_snd_siren,               // 7
    _ambient_snd_fan,                 // 8
    _ambient_snd_spht_door,           // 9
    _ambient_snd_spht_platform,       // 10
    _ambient_snd_heavy_spht_door,     // 11
    _ambient_snd_heavy_spht_platform, // 12
    _ambient_snd_light_machinery,     // 13
    _ambient_snd_heavy_machinery,     // 14
    _ambient_snd_transformer,         // 15
    _ambient_snd_sparking_transformer,// 16
    _ambient_snd_machine_binder,      // 17
    _ambient_snd_machine_bookpress,   // 18
    _ambient_snd_machine_puncher,     // 19
    _ambient_snd_electric,            // 20
    _ambient_snd_alarm,               // 21
    _ambient_snd_night_wind,          // 22
    _ambient_snd_pfhor_door,          // 23
    _ambient_snd_pfhor_platform,      // 24
    _ambient_snd_alien_noise1,        // 25
    _ambient_snd_alien_noise2,        // 26
    _ambient_snd_jjaro_noise,         // 27

    NUMBER_OF_AMBIENT_SOUND_DEFINITIONS
};
```

### Random Sound Codes (`game_sound.h:85-94`)

```c
enum /* random sound codes */
{
    _random_snd_water_drip,
    _random_snd_surface_explosion,
    _random_snd_underground_explosion,
    _random_snd_owl,
    _random_snd_jjaro_creak,

    NUMBER_OF_RANDOM_SOUND_DEFINITIONS
};
```

### Ambient Sound Definition (`sound_definitions.h:53-61`)

```c
struct ambient_sound_definition
{
    short sound_index;
};

struct random_sound_definition
{
    short sound_index;
};
```

### Ambient Sound Pipeline

```
Per-Polygon Ambient Sound:
    1. Check polygon.ambient_sound_image_index
    2. If player in polygon with ambient sound:
       - Start/continue ambient playback on ambient channel
       - Volume based on player position within polygon
    3. If player leaves polygon:
       - Fade out ambient over ~0.5 seconds

Random sounds triggered at intervals defined per-sound
```

---

## 13.11 Sound File Format (`sound_definitions.h:63-80`)

```c
enum
{
    SOUND_FILE_VERSION= 1,
    SOUND_FILE_TAG= 'snd2'
};

struct sound_file_header
{
    long version;
    long tag;

    short source_count;  // usually 2 (8-bit, 16-bit)
    short sound_count;

    short unused[124];

    // immediately followed by source_count*sound_count sound_definition structures
};
```

---

## 13.12 Sound Playback Flow

```
Play Sound Request:
    │
    ├─► Calculate distance from listener
    │
    ├─► Calculate volume based on distance
    │     └─► If volume < ABORT_AMPLITUDE_THRESHHOLD: skip
    │
    ├─► Calculate stereo panning
    │
    ├─► Check for obstruction
    │     └─► Reduce volume if obstructed
    │
    ├─► Find available channel
    │     ├─► Look for free channel
    │     ├─► If none free, check if new sound is louder
    │     └─► If louder, preempt existing sound
    │
    ├─► Select permutation (avoid recent repeats)
    │
    ├─► Apply pitch variation
    │
    └─► Start playback on selected channel
```

---

## 13.13 Dynamic Tracking

For moving sound sources (monsters, projectiles), Marathon updates the sound position each tick.

```c
// In sound_manager_idle_proc():
for (each active channel) {
    if (channel->dynamic_source != NULL) {
        // Source is moving, update position
        recalculate_volume_and_pan(channel);
    }
}
```

This allows sounds to move with their source, creating a more immersive experience.

---

## 13.14 Summary

Marathon's sound system provides spatial audio within 1995 hardware constraints:

**Channel System:**
- 4 normal channels + 2 ambient channels
- Priority-based preemption
- Volume threshold prevents quiet sounds from playing

**3D Positioning:**
- Distance attenuation
- Stereo panning based on angle
- Obstruction detection

**Ambient Sounds:**
- Per-polygon environmental sounds
- Random sounds at intervals
- Crossfading between areas

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `MAXIMUM_SOUND_CHANNELS` | 4 | `game_sound.c:106` |
| `MAXIMUM_AMBIENT_SOUND_CHANNELS` | 2 | `game_sound.c:107` |
| `MAXIMUM_SOUND_VOLUME` | 256 | `game_sound.h:13` |
| `ABORT_AMPLITUDE_THRESHHOLD` | ~42 | `game_sound.c:103` |
| `MAXIMUM_PERMUTATIONS_PER_SOUND` | 5 | `sound_definitions.h:15` |
| `NUMBER_OF_AMBIENT_SOUND_DEFINITIONS` | 28 | `game_sound.h:82` |
| `NUMBER_OF_RANDOM_SOUND_DEFINITIONS` | 5 | `game_sound.h:93` |

### Key Source Files

| File | Purpose |
|------|---------|
| `game_sound.c` | Sound logic, 3D positioning, channel management |
| `game_sound.h` | Constants, enums, public interface |
| `sound_definitions.h` | Sound definition structures, behaviors |
| `sound_macintosh.c` | Mac Sound Manager interface (replace for porting) |
| `ambient_sound.c` | Ambient sound source management |

---

## 13.15 See Also

- [Chapter 7: Game Loop](07_game_loop.md) — `sound_manager_idle_proc()` called each tick
- [Chapter 25: Media](25_media.md) — Media-related sound effects
- [Chapter 10: File Formats](10_file_formats.md) — Sound file structure

---

*Next: [Chapter 14: Items & Inventory](14_items.md) - Pickups, weapons, and powerups*
