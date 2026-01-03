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

The key insight is that game logic and rendering are **not** synchronized 1:1. Game logic always runs at exactly 30 Hz, but rendering can run faster (or slower) than 30 FPS.

**How it works (from `interface.c:536-541` and `marathon2.c:73-149`):**

```c
// Main loop calls update_world() which returns number of ticks elapsed
ticks_elapsed = update_world();  // Can return 0, 1, 2, or more!
if (ticks_elapsed > 0) {
    render_screen(ticks_elapsed);  // Only render if game state changed
}
```

Inside `update_world()`, a **for loop** runs as many game ticks as needed:

```c
// From marathon2.c:105-132
time_elapsed = lowest_time;  // How many ticks we can advance
for (i = 0; i < time_elapsed; ++i)
{
    update_lights();
    update_medias();
    update_platforms();
    update_control_panels();
    update_players();        // Process action_flags for this tick
    move_projectiles();
    move_monsters();
    update_effects();
    // ... more updates ...

    dynamic_world->tick_count += 1;
}
```

**Visual: Fast Machine (60 FPS display, 30 Hz game logic)**

```
Real Time:   0ms      16ms      33ms      50ms      66ms      83ms     100ms
             │        │         │         │         │         │         │
Game Logic:  │        │   ┌─────┴─────┐   │   ┌─────┴─────┐   │   ┌─────┴─────┐
(30 Hz)      │        │   │  Tick 1   │   │   │  Tick 2   │   │   │  Tick 3   │
             │        │   └───────────┘   │   └───────────┘   │   └───────────┘
             │        │                   │                   │                │
Render:      │┌──────┐│┌──────┐     ┌────┐│┌──────┐     ┌────┐│┌──────┐     ┌────┐
(60 FPS)     ││Frame │││Frame │     │Frm │││Frame │     │Frm │││Frame │     │Frm │
             ││  0   │││  1   │     │ 2  │││  3   │     │ 4  │││  5   │     │ 6  │
             │└──────┘│└──────┘     └────┘│└──────┘     └────┘│└──────┘     └────┘
             │ 0 ticks│ 0 ticks   1 tick  │ 0 ticks   1 tick  │ 0 ticks   1 tick
             │ elapsed│ elapsed   elapsed │ elapsed   elapsed │ elapsed   elapsed

             Frame 0-1: No tick ready yet, render same state (ticks_elapsed=0)
             Frame 2: Tick 1 ready, update then render
             Frame 3-4: No new tick, render same state
             Frame 5: Tick 2 ready, update then render
```

**Visual: Slow Machine (15 FPS display, game must catch up)**

```
Real Time:   0ms              66ms             133ms            200ms
             │                │                │                │
             │   Frame took   │   Frame took   │   Frame took   │
             │   66ms (slow!) │   66ms         │   66ms         │
             │                │                │                │
Game Logic:  │  ┌─────────────┴─────────────┐  │                │
(catch up!)  │  │ ┌───────┐ ┌───────┐       │  │                │
             │  │ │Tick 1 │ │Tick 2 │       │  │                │
             │  │ └───────┘ └───────┘       │  │                │
             │  │   Run 2 ticks to catch up │  │                │
             │  └───────────────────────────┘  │                │
             │                ↓                │                │
Render:      │┌───────────────────────────────┐│                │
             ││            Frame 0            ││                │
             │└───────────────────────────────┘│                │
             │                                 │                │
                                              ↓
                                ┌───────────────────────────────┐
                                │ ┌───────┐ ┌───────┐           │
                                │ │Tick 3 │ │Tick 4 │           │
                                │ └───────┘ └───────┘           │
                                │   Run 2 more ticks            │
                                └───────────┬───────────────────┘
                                            ↓
                                ┌───────────────────────────────┐
                                │            Frame 1            │
                                └───────────────────────────────┘

Each frame: update_world() runs multiple ticks in a loop to catch up,
            THEN render_screen() draws the final state once.
```

This decoupling ensures:
- **Determinism**: Game always advances in fixed 33.33ms ticks regardless of frame rate
- **Smoothness**: Fast machines can render at 60+ FPS showing the same game state
- **Resilience**: Slow machines catch up by running multiple ticks before rendering

> **Source:** `interface.c:536-541` for main loop, `marathon2.c:105-132` for tick loop

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
// From player.h:53-66 - Extracting encoded values from action flags
#define GET_ABSOLUTE_YAW(i)      (((i) >> 1) & 0x7F)   // 7 bits at position 1
#define GET_ABSOLUTE_PITCH(i)    (((i) >> 9) & 0x1F)   // 5 bits at position 9
#define GET_ABSOLUTE_POSITION(i) (((i) >> 15) & 0x7F)  // 7 bits at position 15
```

**Individual Action Bits (from `player.h:68-106`):**

```c
// Bits 0-7: Yaw control
_absolute_yaw_mode_bit,    // 0 - enables absolute yaw mode
_turning_left_bit,         // 1 - OR absolute_yaw bit 0
_turning_right_bit,        // 2 - OR absolute_yaw bit 1
_sidestep_dont_turn_bit,   // 3 - OR absolute_yaw bit 2
_looking_left_bit,         // 4 - OR absolute_yaw bit 3
_looking_right_bit,        // 5 - OR absolute_yaw bit 4
_absolute_yaw_bit0,        // 6 - OR absolute_yaw bit 5
_absolute_yaw_bit1,        // 7 - OR absolute_yaw bit 6

// Bits 8-13: Pitch control
_absolute_pitch_mode_bit,  // 8 - enables absolute pitch mode
_looking_up_bit,           // 9 - OR absolute_pitch bit 0
_looking_down_bit,         // 10 - OR absolute_pitch bit 1
_looking_center_bit,       // 11 - OR absolute_pitch bit 2
_absolute_pitch_bit0,      // 12 - OR absolute_pitch bit 3
_absolute_pitch_bit1,      // 13 - OR absolute_pitch bit 4

// Bits 14-21: Position control
_absolute_position_mode_bit, // 14 - enables absolute position mode
_moving_forward_bit,         // 15 - OR absolute_position bit 0
_moving_backward_bit,        // 16 - OR absolute_position bit 1
_run_dont_walk_bit,          // 17 - OR absolute_position bit 2
_look_dont_turn_bit,         // 18 - OR absolute_position bit 3
_absolute_position_bit0,     // 19 - OR absolute_position bit 4
_absolute_position_bit1,     // 20 - OR absolute_position bit 5
_absolute_position_bit2,     // 21 - OR absolute_position bit 6

// Bits 22-31: Actions
_sidestepping_left_bit,      // 22
_sidestepping_right_bit,     // 23
_left_trigger_state_bit,     // 24
_right_trigger_state_bit,    // 25
_action_trigger_state_bit,   // 26
_cycle_weapons_forward_bit,  // 27
_cycle_weapons_backward_bit, // 28
_toggle_map_bit,             // 29
_microphone_button_bit,      // 30
_swim_bit                    // 31
```

### Bit Layout Diagram

```
32-bit Action Flags (from player.h:68-106):

┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│31│30│29│28│27│26│25│24│23│22│21│20│19│18│17│16│15│14│13│12│11│10│ 9│ 8│ 7│ 6│ 5│ 4│ 3│ 2│ 1│ 0│
├──┴──┴──┴──┴──┴──┴──┴──┴──┴──┼──┴──┴──┴──┴──┴──┴──┼──┼──┴──┴──┴──┴──┼──┼──┴──┴──┴──┴──┴──┴──┼──┤
│sw│mi│mp│←w│→w│ac│r │l │→s│←s│p6│p5│p4│p3│p2│p1│p0│pM│t4│t3│t2│t1│t0│tM│y6│y5│y4│y3│y2│y1│y0│yM│
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
 │                          │  │                 │  │              │  │                       │
 │        Actions           │  │ Position (7b)   │  │ Pitch (5b)   │  │     Yaw (7 bits)      │
 │     (10 bits)            │  │ or move flags   │  │ or look flgs │  │   or turn/look flags  │
 └──────────────────────────┘  └────────┬────────┘  └──────┬───────┘  └───────────┬───────────┘
                                        │                  │                      │
                                       pM=14              tM=8                   yM=0
                                    (mode bit)         (mode bit)             (mode bit)

Legend:
  yM = absolute_yaw_mode (bit 0)       y0-y6 = yaw value OR turn/look flags
  tM = absolute_pitch_mode (bit 8)     t0-t4 = pitch value OR look up/down/center
  pM = absolute_position_mode (bit 14) p0-p6 = position value OR forward/backward/run
  ←s/→s = sidestep left/right          l/r = left/right trigger
  ac = action trigger                  ←w/→w = cycle weapons backward/forward
  mp = toggle map                      mi = microphone
  sw = swim

Mode bits select interpretation:
  When yM=1: bits 1-7 are 7-bit absolute yaw angle
  When yM=0: bits 1-7 are individual turn/look boolean flags
  (Same pattern for pitch and position)

Result: Full 6-DOF input in just 32 bits!
```

> **Source:** `player.h:53-66` for GET macros, `player.h:68-106` for bit enum

### Converting Input to Action Flags

```c
uint32_t encode_action_flags(platform_input_state* input) {
    uint32_t flags = 0;

    // Movement booleans (when NOT using absolute position mode)
    if (input->forward)         flags |= _moving_forward;
    if (input->backward)        flags |= _moving_backward;
    if (input->strafe_left)     flags |= _sidestepping_left;
    if (input->strafe_right)    flags |= _sidestepping_right;
    if (input->run)             flags |= _run_dont_walk;

    // Look direction booleans (when NOT using absolute yaw/pitch mode)
    if (input->turn_left)       flags |= _turning_left;
    if (input->turn_right)      flags |= _turning_right;
    if (input->look_up)         flags |= _looking_up;
    if (input->look_down)       flags |= _looking_down;

    // Action buttons
    if (input->fire_primary)    flags |= _left_trigger_state;
    if (input->fire_secondary)  flags |= _right_trigger_state;
    if (input->action)          flags |= _action_trigger_state;

    // Encode absolute angles (for mouse look) - sets mode bit + value
    // Uses SET_ABSOLUTE_YAW/PITCH/POSITION macros from player.h
    if (input->use_absolute_yaw) {
        flags |= _absolute_yaw_mode;                    // Set mode bit 0
        flags |= (input->yaw & 0x7F) << 1;              // 7-bit yaw at bits 1-7
    }
    if (input->use_absolute_pitch) {
        flags |= _absolute_pitch_mode;                  // Set mode bit 8
        flags |= (input->pitch & 0x1F) << 9;            // 5-bit pitch at bits 9-13
    }

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
