# Chapter 3: Engine Overview

## High-Level Architecture and Subsystem Interactions

---

## 3.1 Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│              Shell / Main Loop (30 Hz)              │
│            (1 tick = 1/30th sec = 33.33ms)          │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
      ┌────▼────┐                ┌───▼────┐
      │  Input  │                │Network │
      │ System  │                │  Sync  │
      └────┬────┘                └───┬────┘
           │                         │
           └────────┬────────────────┘
                    │
      ┌─────────────▼─────────────┐
      │     World Update          │
      │  (all systems, per tick)  │
      │  • Lights                 │
      │  • Media                  │
      │  • Platforms              │
      │  • Players                │
      │  • Projectiles            │
      │  • Monsters               │
      │  • Effects                │
      └─────────────┬─────────────┘
                    │
           ┌────────▼────────┐
           │  Render Engine  │
           │ (Portal-Based)  │
           └─────────────────┘
```

> **Timing note:** The world update runs in a loop N times per frame, where N = elapsed ticks since last update. If 1/30th sec elapsed → runs once. If 1/15th sec elapsed (slow frame) → runs twice to catch up. This ensures deterministic 30 Hz simulation regardless of frame rate. See [Chapter 7.6](07_game_loop.md#76-tick-accumulation-and-catch-up) for the catch-up mechanism.

---

## 3.2 Subsystem Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              MAIN LOOP (shell.c)                                │
│                           Called each frame                                     │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│    INPUT      │           │   NETWORK     │           │    AUDIO      │
│  vbl.c        │           │  network.c    │           │ game_sound.c  │
│  action_flags │──────────▶│  sync actions │           │ 3D positioned │
└───────────────┘           └───────┬───────┘           └───────▲───────┘
                                    │                           │
                                    ▼                           │
┌───────────────────────────────────────────────────────────────┼─────────────────┐
│                         WORLD UPDATE (marathon2.c)            │                 │
│                   update_world() - loops N times              │                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │ UPDATE ORDER (repeated N times, once per elapsed tick):                 │    │
│  │  1. update_lights()      → lightsource.c                                │    │
│  │  2. update_medias()      → media.c (water/lava levels)                  │    │
│  │  3. update_platforms()   → platforms.c (doors/elevators)                │    │
│  │  4. update_players()     → player.c + physics.c                         │    │
│  │  5. move_projectiles()   → projectiles.c                                │    │
│  │  6. move_monsters()      → monsters.c + pathfinding.c                   │    │
│  │  7. update_effects()     → effects.c (particles/explosions)             │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           RENDERING (render.c)                                  │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌────────────┐   │
│  │   Portal    │───▶│   Polygon    │───▶│    Texture      │───▶│  Screen    │   │
│  │   Culling   │    │   Clipping   │    │    Mapping      │    │  Output    │   │
│  │  (map.c)    │    │  (render.c)  │    │(scottish_tex.c) │    │ (screen.c) │   │
│  └─────────────┘    └──────────────┘    └─────────────────┘    └────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

> **How the loop works:** `update_world()` calculates N = ticks elapsed since last call, then runs the update sequence N times. This is the catch-up mechanism from [Chapter 7.6](07_game_loop.md#76-tick-accumulation-and-catch-up). In the source: `for (i=0; i<time_elapsed; ++i) { update_lights(); ... }` (`marathon2.c:105-132`).

---

## 3.3 Data Flow Diagram

```
                    ┌─────────────────────────────────────┐
                    │          FILE SYSTEM                │
                    │  ┌─────────┐ ┌─────────┐ ┌───────┐  │
                    │  │Map WAD  │ │Shapes16 │ │Sounds │  │
                    │  └────┬────┘ └────┬────┘ └───┬───┘  │
                    └───────┼───────────┼──────────┼──────┘
                            │           │          │
                            ▼           ▼          ▼
                    ┌───────────┐ ┌──────────┐ ┌─────────────┐
                    │  wad.c    │ │ shapes.c │ │game_sound.c │
                    │game_wad.c │ │          │ │             │
                    └─────┬─────┘ └────┬─────┘ └──────┬──────┘
                          │            │              │
                          ▼            ▼              ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          RUNTIME DATA STRUCTURES                           │
│                                                                            │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │  map_polygons│  │ map_endpoints │  │  objects[]  │  │  players[]     │  │
│  │  [1024 max]  │  │ [2048 max]    │  │  [384 max]  │  │  [8 max]       │  │
│  └──────────────┘  └───────────────┘  └─────────────┘  └────────────────┘  │
│                                                                            │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │  monsters[]  │  │ projectiles[] │  │  effects[]  │  │  platforms[]   │  │
│  │  [220 max]   │  │ [32 max]      │  │  [64 max]   │  │  [64 max]      │  │
│  └──────────────┘  └───────────────┘  └─────────────┘  └────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 3.4 Subsystem Quick Reference

| Subsystem | Primary Files | Purpose | Chapter |
|-----------|---------------|---------|---------|
| **World/Map** | map.c, world.c | Polygon geometry | [4](04_world.md) |
| **Rendering** | render.c, scottish_textures.c | Portal culling, textures | [5](05_rendering.md) |
| **Physics** | physics.c | Movement, collision | [6](06_physics.md) |
| **Game Loop** | marathon2.c | 30 Hz update | [7](07_game_loop.md) |
| **Entities** | monsters.c, weapons.c, projectiles.c | AI, combat | [8](08_entities.md) |
| **Network** | network.c | Peer-to-peer sync | [9](09_networking.md) |
| **Files** | wad.c, game_wad.c | WAD format | [10](10_file_formats.md) |
| **Sound** | game_sound.c | 3D audio | [13](13_sound.md) |
| **Items** | items.c | Pickups | [14](14_items.md) |
| **Panels** | control_panels.c | Switches, terminals | [15](15_control_panels.md) |

---

## 3.5 Coordinate System

Marathon uses a right-handed coordinate system with fixed-point values.

### World Coordinates

```
                        Top-Down View (X-Y Plane)

                              +Y (North)
                                ↑
                                │
                                │
                 ───────────────┼───────────────→ +X (East)
                                │
                                │
                              -Y (South)

                        +Z is UP (out of page)
                        -Z is DOWN (into page)
```

### Unit System

From `world.h:30-41`:

**World Units** (10 fractional bits):
```c
typedef short world_distance;  // 16-bit signed (line 41)
#define WORLD_ONE 1024         // 1.0 in world units (line 31)
```

**Fixed-Point** (16 fractional bits):
```c
typedef long fixed;            // 32-bit signed (cseries.h:122)
#define FIXED_ONE 65536        // 1.0 in fixed-point (cseries.h:110)
```

### Scale Reference

| Measurement | World Units |
|-------------|-------------|
| Player height | ~819 (0.8 WU) |
| Player radius | ~256 (0.25 WU) |
| Door width | ~1024-2048 |
| Typical room | ~4096-8192 |
| Max step-up | ~341 (1/3 WU) |

---

## 3.6 Angle System

From `world.h:22-28`:
```c
typedef short angle;  // 16-bit, but only 9 bits used

#define NUMBER_OF_ANGLES 512    // Full circle (line 22)
#define HALF_CIRCLE 256         // 180° (line 25)
#define QUARTER_CIRCLE 128      // 90° (line 24)
```

**Visualization:**

```
              128 (North/+Y)
                     ↑
                     │
       192 ──────────┼──────────→ 0 (East/+X)
       (West)        │
                     │
              384 (South/-Y)

Angles increase counter-clockwise
```

---

## 3.7 Summary

Marathon's engine is built on clear architectural principles:

**Main Loop:**
- Fixed 30 Hz timestep
- Deterministic update order
- Rendering decoupled from logic

**Key Systems:**
- World/Map: Polygon-based geometry
- Rendering: Portal visibility culling
- Physics: Fixed-point collision
- Entities: State machines for AI
- Network: Deterministic sync

**Data Flow:**
- Files loaded at level start
- Runtime arrays for entities
- All state deterministic

---

*Next: [Chapter 4: World Representation](04_world.md) - Polygons, lines, and map structure*
