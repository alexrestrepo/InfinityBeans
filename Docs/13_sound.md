# Chapter 13: Sound System

## 3D Audio, Channels, and Ambient Sounds

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
  4. ABORT_AMPLITUDE_THRESHOLD prevents quiet sounds from interrupting
```

---

## 13.3 Core Constants

```c
#define MAXIMUM_SOUND_CHANNELS 4         // Normal sound channels
#define MAXIMUM_AMBIENT_SOUND_CHANNELS 2 // Background/environmental
#define MAXIMUM_SOUND_VOLUME 256         // Full volume
#define NUMBER_OF_SOUND_VOLUME_LEVELS 8  // User preference levels
#define ABORT_AMPLITUDE_THRESHOLD (MAXIMUM_SOUND_VOLUME/6)  // ~42
#define MINIMUM_RESTART_TICKS (MACHINE_TICKS_PER_SECOND/12) // ~5 ticks
```

### Initialization Flags

| Flag | Value | Description |
|------|-------|-------------|
| `_stereo_flag` | 0x0001 | Enable stereo panning |
| `_dynamic_tracking_flag` | 0x0002 | Track moving sound sources |
| `_doppler_shift_flag` | 0x0004 | Pitch shift based on velocity |
| `_ambient_sound_flag` | 0x0008 | Enable ambient sounds |
| `_16bit_sound_flag` | 0x0010 | Use 16-bit audio |
| `_more_sounds_flag` | 0x0020 | Load additional sound variations |
| `_extra_memory_flag` | 0x0040 | Use extra memory for sound cache |

---

## 13.4 Channel Data Structure

```c
struct channel_data {
    word flags;                      // Channel state flags
    short sound_index;               // Currently playing sound
    short identifier;                // Unique sound instance ID
    struct sound_variables variables; // Volume, pitch, etc.
    world_location3d *dynamic_source; // Moving source (tracked)
    world_location3d source;          // Static source position
    unsigned long start_tick;         // When playback started
};
```

---

## 13.5 3D Sound Positioning

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

## 13.6 Sound Obstruction

Marathon checks for walls and liquid surfaces between listener and source.

### Obstruction Flags

| Flag | Value | Effect |
|------|-------|--------|
| `_sound_was_obstructed` | 0x0001 | Wall between source and listener |
| `_sound_was_media_obstructed` | 0x0002 | Liquid surface between source and listener |
| `_sound_was_media_muffled` | 0x0004 | Both in liquid but separated |

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

## 13.7 Ambient Sound System

Ambient sounds provide environmental atmosphere based on the player's location.

### Ambient Sound Types

| ID | Name | Loop Type |
|----|------|-----------|
| 0 | `_ambient_snd_water` | Continuous |
| 1 | `_ambient_snd_sewage` | Continuous |
| 2 | `_ambient_snd_lava` | Continuous |
| 3 | `_ambient_snd_goo` | Continuous |
| 4 | `_ambient_snd_under_media` | Continuous |
| 5 | `_ambient_snd_wind` | Continuous |
| 6 | `_ambient_snd_waterfall` | Continuous |
| 7 | `_ambient_snd_siren` | Continuous |
| 8 | `_ambient_snd_fan` | Continuous |
| 9 | `_ambient_snd_spht_door` | Continuous |
| 10 | `_ambient_snd_spht_platform` | Continuous |

### Random Sound Types

| ID | Name | Trigger |
|----|------|---------|
| 0 | `_random_snd_water_drip` | Random interval |
| 1 | `_random_snd_surface_explosion` | Random interval |
| 2 | `_random_snd_underground_explosion` | Random interval |
| 3 | `_random_snd_owl` | Random interval |
| 4 | `_random_snd_creak` | Random interval |

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

## 13.8 Sound Permutations

Marathon supports multiple variations of each sound to prevent repetition.

```c
struct sound_definition {
    short sound_code;           // Unique identifier
    short behavior_index;       // How sound behaves (looping, etc.)
    word flags;                 // Various flags
    word chance;                // Random playback chance [0, 65535]
    fixed low_pitch, high_pitch; // Pitch variation range
    short permutations;         // Number of variations
    short permutations_played;  // Bitmask of recently played
    // ... followed by permutation data
};

// Example: Rifle fire has 3 permutations
// Each play randomly selects one (avoiding recent repeats)
```

---

## 13.9 Sound Playback Flow

```
Play Sound Request:
    │
    ├─► Calculate distance from listener
    │
    ├─► Calculate volume based on distance
    │     └─► If volume < ABORT_AMPLITUDE_THRESHOLD: skip
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

## 13.10 Dynamic Tracking

For moving sound sources (monsters, projectiles), Marathon updates the sound position each tick.

```c
// In sound update loop:
for (each active channel) {
    if (channel->dynamic_source != NULL) {
        // Source is moving, update position
        recalculate_volume_and_pan(channel);
    }
}
```

This allows sounds to move with their source, creating a more immersive experience.

---

## 13.11 Summary

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

**For Porting:**
- Replace Mac Sound Manager with miniaudio or similar
- Keep channel management logic from `game_sound.c`
- Sound data is standard PCM in Sounds files

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAXIMUM_SOUND_CHANNELS` | 4 | Normal sound channels |
| `MAXIMUM_AMBIENT_SOUND_CHANNELS` | 2 | Environmental channels |
| `MAXIMUM_SOUND_VOLUME` | 256 | Full volume |
| `ABORT_AMPLITUDE_THRESHOLD` | ~42 | Minimum playback volume |

### Key Source Files

| File | Purpose |
|------|---------|
| `game_sound.c` | Sound logic, 3D positioning |
| `sound_macintosh.c` | Mac Sound Manager (replace) |
| `ambient_sound.c` | Ambient sound management |

---

*Next: [Chapter 14: Items & Inventory](14_items.md) - Pickups, weapons, and powerups*
