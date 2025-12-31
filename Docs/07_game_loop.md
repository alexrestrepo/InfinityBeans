# Chapter 7: Game Loop and Timing

## The Heartbeat of Marathon

> **For Porting:** `marathon2.c` contains the main game loop and is mostly portable. Replace the Mac event handling in `shell.c` with your window system's event loop. The core `update_world()` function needs no changes—just call it at 30 Hz with proper action flags.

---

## 7.1 What Problem Are We Solving?

Marathon needs to orchestrate all game systems in a coordinated, predictable manner. This includes:

- **Input processing** - Reading keyboard and mouse state
- **Physics updates** - Moving players, monsters, projectiles
- **World simulation** - Platforms, lights, liquids, effects
- **Rendering** - Drawing the current game state
- **Networking** - Synchronizing multiplayer games
- **Recording/Replay** - Capturing and playing back game sessions

**The constraints:**
- Must run at exactly **30 ticks per second** (deterministic timing)
- Same inputs must produce same outputs (critical for networking)
- All systems must update in the correct order
- Must handle variable rendering frame rates

**Marathon's solution: Fixed-Timestep Game Loop**

Every 33.33 milliseconds (1/30th of a second), Marathon runs one complete game tick. Physics, AI, and game logic all advance by exactly one tick. This fixed timestep ensures deterministic behavior regardless of machine speed.

---

## 7.2 Understanding the Fixed Timestep

Before diving into code, let's understand why Marathon uses a fixed timestep.

### The Problem with Variable Timestep

Many games use variable timestep—updating based on how much real time passed:

```
Variable Timestep (problematic for networking):

Frame 1: dt = 16ms  → physics.update(0.016)
Frame 2: dt = 33ms  → physics.update(0.033)  (slow frame)
Frame 3: dt = 17ms  → physics.update(0.017)

Problem: Different machines run at different speeds
         Results are NOT identical!
         Multiplayer desynchronizes!
```

### The Fixed Timestep Solution

Marathon always advances exactly one tick per update:

```
Fixed Timestep (Marathon's approach):

Tick 1: physics.update(1/30)  ← Always exactly 1/30 second
Tick 2: physics.update(1/30)  ← Same delta every time
Tick 3: physics.update(1/30)  ← Deterministic!

Result: Same inputs → same outputs
        All machines stay synchronized
        Replays work perfectly
```

### Decoupling Update and Render

```
The key insight: Game logic and rendering run at DIFFERENT rates

Game Logic:     ┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐
                │Tick │     │Tick │     │Tick │     │Tick │
                │  0  │     │  1  │     │  2  │     │  3  │
                └──┬──┘     └──┬──┘     └──┬──┘     └──┬──┘
                   │           │           │           │
Time:    0ms     33ms        66ms       100ms      133ms
                   │           │           │           │
Rendering:     ┌───┴───┐   ┌──┴──┐   ┌──┴──┐   ┌──┴──┐
(60 FPS)       │Frame 0│   │Fr 1 │   │Fr 2 │   │Fr 3 │...
               │       │   │     │   │     │   │     │
               └───────┘   └─────┘   └─────┘   └─────┘
                 16ms        16ms      16ms      16ms

Rendering can happen faster than game updates!
Multiple renders can show the same game state
```

---

## 7.3 Let's Build: A Simple Game Loop

Before seeing Marathon's full implementation, let's build a simplified version.

### Step 1: Basic Fixed Timestep Loop

```
PSEUDOCODE: Simple fixed timestep

TICK_DURATION = 33.33ms  (1/30th second)

function game_loop():
    last_update_time = current_time()
    accumulated_time = 0

    while game_running:
        current = current_time()
        frame_time = current - last_update_time
        last_update_time = current

        accumulated_time += frame_time

        -- Run game ticks until caught up
        while accumulated_time >= TICK_DURATION:
            input = read_input()
            update_game_one_tick(input)
            accumulated_time -= TICK_DURATION

        -- Render current state (can be faster than 30 FPS)
        render_frame()
```

Modern C implementation:

```c
#define TICKS_PER_SECOND 30
#define TICK_MS (1000 / TICKS_PER_SECOND)  // 33ms

void game_loop(void) {
    uint32_t last_tick_time = platform_get_ticks();
    int32_t accumulated_ms = 0;

    while (game_running) {
        uint32_t current_time = platform_get_ticks();
        uint32_t frame_time = current_time - last_tick_time;
        last_tick_time = current_time;

        accumulated_ms += frame_time;

        // Run game logic at fixed 30 Hz
        while (accumulated_ms >= TICK_MS) {
            uint32_t action_flags = read_input();
            update_world_one_tick(action_flags);
            accumulated_ms -= TICK_MS;
        }

        // Render can run faster than game logic
        render_frame();
    }
}
```

### Step 2: The Update Order

The order of updates matters! Dependencies must be respected:

```c
void update_world_one_tick(uint32_t action_flags) {
    // 1. Input affects player physics
    process_player_input(action_flags);

    // 2. Environment updates (may affect entities)
    update_lights();
    update_platforms();     // Doors, elevators
    update_liquids();       // Water levels, currents

    // 3. Entity updates (may spawn projectiles)
    update_players();       // Player physics
    update_monsters();      // AI, movement, attacks
    update_projectiles();   // Bullet paths
    update_effects();       // Particles, explosions

    // 4. World maintenance
    respawn_items();
    play_ambient_sounds();
    check_level_completion();

    // 5. Record for replay
    record_action_flags(action_flags);
}
```

**Marathon's approach:** Marathon uses exactly this structure, with careful ordering to ensure determinism. The key insight is that all physics happens within the same tick, so there are no race conditions between systems.

---

## 7.4 Marathon's Main Loop

Now let's see Marathon's actual implementation.

### The 30 Hz Tick

**Fundamental Constant:**
```c
#define TICKS_PER_SECOND 30
```

This means:
- Each tick is exactly 33.33 milliseconds
- All physics calculations assume this fixed duration
- Network synchronization depends on it

### The Update Order (update_world)

Each tick, Marathon updates systems in this precise order:

```
Main Loop (update_world()):

1. Input Processing - Gather action flags
2. For each queued tick:
   a. update_lights()           - Animate lighting
   b. update_medias()           - Liquid level, damage
   c. update_platforms()        - Elevators, doors
   d. update_control_panels()   - Terminals, switches
   e. update_players()          - Player physics
   f. move_projectiles()        - Projectile paths
   g. move_monsters()           - AI, attacks
   h. update_effects()          - Particles, explosions
   i. recreate_objects()        - Respawn items
   j. handle_random_sound_image() - Ambient sounds
   k. animate_scenery()         - Static objects
   l. update_net_game()         - Network sync
   m. Check level completion
3. Render Update - HUD, interface
4. Record/Replay - Store action flags
```

### Why This Order Matters

```
Update Order Dependencies:

┌─────────────────────────────────────────────────────────────────┐
│  ENVIRONMENT UPDATES (affect everything else)                   │
│                                                                 │
│  update_lights()                                                │
│       │                                                         │
│       └──► Affects shading tables for rendering                 │
│            Affects media height (lights control water level!)   │
│                                                                 │
│  update_medias()                                                │
│       │                                                         │
│       └──► Changes floor heights (player physics needs this)    │
│            Applies damage (player/monster health)               │
│                                                                 │
│  update_platforms()                                             │
│       │                                                         │
│       └──► Moves floors/ceilings                                │
│            Can crush entities                                   │
│            Changes polygon connectivity                         │
├─────────────────────────────────────────────────────────────────┤
│  ENTITY UPDATES (depend on environment)                         │
│                                                                 │
│  update_players()                                               │
│       │                                                         │
│       └──► Physics uses current floor heights                   │
│            Can trigger platforms/switches                       │
│            Can fire weapons (spawns projectiles)                │
│                                                                 │
│  move_projectiles()                                             │
│       │                                                         │
│       └──► Uses current polygon geometry                        │
│            Can damage monsters/players                          │
│            Can spawn effects                                    │
│                                                                 │
│  move_monsters()                                                │
│       │                                                         │
│       └──► Uses current player positions                        │
│            Can fire projectiles                                 │
│            Can be damaged by projectiles from this tick         │
├─────────────────────────────────────────────────────────────────┤
│  MAINTENANCE (cleanup and spawning)                             │
│                                                                 │
│  update_effects()     - Remove expired effects                  │
│  recreate_objects()   - Respawn items                           │
│  update_net_game()    - Sync with other players                 │
│  check_completion()   - Level end conditions                    │
└─────────────────────────────────────────────────────────────────┘
```

### Critical Design Principle

All physics updates happen within the same tick, then rendering occurs. This ensures:
- **Network consistency** - Same inputs → same state on all machines
- **Deterministic replay** - Recorded games play back identically
- **No visual glitches** - Rendering sees consistent world state

---

## 7.5 Action Flags (Input Encoding)

Marathon compresses all player input into a single 32-bit value called **action flags**.

### Why Compress Input?

```
Network Bandwidth Optimization:

Naive approach (sending all input state):
  - Position (3 × 32-bit floats) = 12 bytes
  - Velocity (3 × 32-bit floats) = 12 bytes
  - Angles (3 × 32-bit floats) = 12 bytes
  - Buttons (8+ booleans) = 8 bytes
  Total: 44+ bytes per player per tick

Marathon's approach (action flags):
  - One 32-bit integer = 4 bytes per player per tick

Bandwidth comparison (4 players, 30 ticks/sec):
  Naive:     4 × 44 × 30 = 5,280 bytes/sec
  Marathon:  4 × 4 × 30 = 480 bytes/sec

10× bandwidth reduction!
```

### The 32-bit Action Flags Layout

```c
// Extracting encoded values from action flags
#define GET_ABSOLUTE_YAW(flags)       ((flags >> 7) & 0x7F)
#define GET_ABSOLUTE_PITCH(flags)     ((flags >> 14) & 0x1F)
#define GET_ABSOLUTE_POSITION(flags)  ((flags >> 22) & 0x7F)
```

**Individual Action Bits:**

```c
// Boolean flags encoded in lower bits
_turning_left, _turning_right
_looking_left, _looking_right
_looking_up, _looking_down, _looking_center
_moving_forward, _moving_backward
_sidestepping_left, _sidestepping_right
_run_dont_walk
_left_trigger_state, _right_trigger_state
_action_trigger_state
_cycle_weapons_forward, _cycle_weapons_backward
_toggle_map, _swim
```

### Bit Layout Diagram

```
32-bit Action Flags:

Bits 0-6:   Absolute yaw (7 bits)        → 128 angles
Bits 7-13:  Reserved/flags (7 bits)
Bits 14-18: Absolute pitch (5 bits)      → 32 angles
Bits 19-21: Reserved/flags (3 bits)
Bits 22-28: Absolute position (7 bits)   → Position encoding
Bits 29-31: Movement flags (3 bits)

┌─────────────────────────────────────────────────────────────┐
│ 31 30 29 │ 28 27 26 25 24 23 22 │ 21 20 19 │ 18 17 16 15 14 │
├──────────┼─────────────────────┼──────────┼────────────────┤
│ Movement │   Absolute Position  │ Reserved │ Absolute Pitch │
│  flags   │      (7 bits)        │ (3 bits) │   (5 bits)     │
└──────────┴─────────────────────┴──────────┴────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 13 12 11 10 9 8 7 │ 6 5 4 3 2 1 0 │
├───────────────────┼───────────────┤
│  Reserved/Flags   │ Absolute Yaw  │
│    (7 bits)       │   (7 bits)    │
└───────────────────┴───────────────┘

Result: Full 6-DOF input in just 32 bits!
```

### Converting Input to Action Flags

```c
uint32_t encode_action_flags(platform_input_state* input) {
    uint32_t flags = 0;

    // Movement booleans
    if (input->forward)         flags |= _moving_forward;
    if (input->backward)        flags |= _moving_backward;
    if (input->strafe_left)     flags |= _sidestepping_left;
    if (input->strafe_right)    flags |= _sidestepping_right;
    if (input->run)             flags |= _run_dont_walk;

    // Look direction booleans
    if (input->turn_left)       flags |= _turning_left;
    if (input->turn_right)      flags |= _turning_right;
    if (input->look_up)         flags |= _looking_up;
    if (input->look_down)       flags |= _looking_down;

    // Action buttons
    if (input->fire_primary)    flags |= _left_trigger_state;
    if (input->fire_secondary)  flags |= _right_trigger_state;
    if (input->action)          flags |= _action_trigger_state;

    // Encode absolute angles (for mouse look)
    flags |= (input->yaw & 0x7F) << 0;      // 7-bit yaw
    flags |= (input->pitch & 0x1F) << 14;   // 5-bit pitch

    return flags;
}
```

### Decoding Action Flags

```c
void decode_action_flags(uint32_t flags, player_input* out) {
    // Extract booleans
    out->moving_forward = (flags & _moving_forward) != 0;
    out->moving_backward = (flags & _moving_backward) != 0;
    out->strafing_left = (flags & _sidestepping_left) != 0;
    out->strafing_right = (flags & _sidestepping_right) != 0;
    out->running = (flags & _run_dont_walk) != 0;

    // Extract angles
    out->absolute_yaw = GET_ABSOLUTE_YAW(flags);
    out->absolute_pitch = GET_ABSOLUTE_PITCH(flags);

    // Extract triggers
    out->primary_trigger = (flags & _left_trigger_state) != 0;
    out->secondary_trigger = (flags & _right_trigger_state) != 0;
    out->action = (flags & _action_trigger_state) != 0;
}
```

---

## 7.6 Tick Accumulation and Catch-Up

Marathon handles slow machines by running multiple ticks when needed.

### The Accumulator Pattern

```
Scenario: Machine runs slow, misses a tick

Real Time:    0ms    33ms    66ms    100ms   133ms
              │      │       │       │       │
              ▼      ▼       ▼       ▼       ▼
Ideal Ticks:  T0     T1      T2      T3      T4

Actual:       T0     (slow)  T1+T2   T3      T4
                     │       │
                     │       └─► Run 2 ticks to catch up!
                     │
                     └─► Frame took 50ms instead of 33ms

Accumulator tracks missed time:
  Frame 1: accumulator = 33ms, run 1 tick, accumulator = 0ms
  Frame 2: accumulator = 50ms, run 1 tick, accumulator = 17ms
  Frame 3: accumulator = 17ms + 33ms = 50ms, run 1 tick, accumulator = 17ms
  Frame 4: accumulator = 17ms + 33ms = 50ms, run 1 tick, accumulator = 17ms
  ...
  Eventually: accumulator = 33ms+, run 2 ticks to catch up
```

### Implementation

```c
void main_loop(void) {
    uint32_t last_time = platform_get_ticks();
    int32_t accumulated = 0;

    while (game_running) {
        uint32_t current = platform_get_ticks();
        int32_t elapsed = current - last_time;
        last_time = current;

        accumulated += elapsed;

        // Cap accumulated time to prevent spiral of death
        if (accumulated > TICK_MS * 10) {
            accumulated = TICK_MS * 10;  // Max 10 ticks per frame
        }

        // Run as many ticks as needed
        int ticks_run = 0;
        while (accumulated >= TICK_MS) {
            uint32_t action_flags = get_action_flags();
            update_world_one_tick(action_flags);
            accumulated -= TICK_MS;
            ticks_run++;
        }

        // Always render (even if no ticks run)
        render_frame();
    }
}
```

### The Spiral of Death

```
Problem: If ticks take longer than 33ms, you fall further behind

Tick takes 40ms:
  Frame 1: Run 1 tick (40ms), now 7ms behind
  Frame 2: Run 1 tick (40ms), now 14ms behind
  Frame 3: Run 2 ticks (80ms), now 47ms behind
  Frame 4: Run 2 ticks (80ms), now 80ms behind
  ...game grinds to a halt...

Solution: Cap the accumulator
  If more than 10 ticks behind, discard extra time
  Game runs slower but doesn't freeze
```

---

## 7.7 Summary

Marathon's game loop is built around a fixed 30 Hz timestep that ensures deterministic behavior:

**Fixed Timestep:**
- Exactly 30 ticks per second (33.33ms each)
- All physics and game logic use this constant
- Critical for networking and replays

**Update Order:**
1. Environment (lights, platforms, liquids)
2. Entities (players, monsters, projectiles)
3. Maintenance (respawning, sound, level checks)
4. Network sync and recording

**Action Flags:**
- All player input compressed to 32 bits
- Enables low-bandwidth networking (480 bytes/sec for 4 players)
- Includes movement, look direction, triggers, and weapons

**Timing:**
- Accumulator pattern handles variable frame times
- Can run multiple ticks per frame to catch up
- Capped to prevent spiral of death

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `TICKS_PER_SECOND` | 30 | Game update rate |
| Tick duration | 33.33ms | Time per tick |
| Accumulator cap | 10 ticks | Prevent spiral of death |

### Key Source Files

| File | Purpose |
|------|---------|
| `marathon2.c` | Main game loop and state machine |
| `shell.c` | Mac event loop (replace for porting) |
| `vbl.c` | Timing and input coordination |
| `player.c` | Player input processing |
| `world.c` | World update orchestration |

### Source Reference Summary

| Function/Structure | Location |
|-------------------|----------|
| `update_world()` | marathon2.c |
| `TICKS_PER_SECOND` | constants.h |
| Action flag macros | player.h |

---

*Next: [Chapter 8: Entity Systems](08_entities.md) - Monsters, weapons, projectiles, and effects*
