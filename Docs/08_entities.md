# Chapter 8: Entity Systems

## Monsters, Weapons, and Combat

> **For Porting:** All entity code is fully portable! `monsters.c`, `projectiles.c`, `weapons.c`, `items.c`, `effects.c`, and `scenery.c` have no Mac dependencies. The definition headers (`*_definitions.h`) contain static data tables that compile anywhere.

---

## 8.1 What Problem Are We Solving?

Marathon needs to simulate a living world filled with:

- **Monsters** - 47 types with AI, pathfinding, and combat behaviors
- **Weapons** - 10 player weapons with different fire modes
- **Projectiles** - 39 projectile types with unique physics
- **Effects** - Visual feedback (explosions, sparks, blood)
- **Items** - Pickups, ammunition, health

**The constraints:**
- Must support many simultaneous entities (monsters, projectiles, effects)
- AI must be responsive but not consume excessive CPU
- All behavior must be deterministic (for networking)
- Visual effects must provide clear combat feedback

**Marathon's solution: State Machine Architecture**

Each entity type uses a state machine that advances deterministically each tick. Complex behaviors emerge from simple state transitions combined with pathfinding and physics systems.

---

## 8.2 Understanding the Entity Architecture

Before diving into specific systems, let's understand how Marathon organizes entities.

### Entity Storage

Marathon uses arrays with linked lists for efficient access:

```
Entity Organization:

┌─────────────────────────────────────────────────────────────────┐
│                     OBJECTS ARRAY                               │
│  Global storage for all world objects (monsters, items, etc.)   │
│                                                                 │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐      │
│  │Obj 0 │Obj 1 │Obj 2 │ ...  │ ...  │ ...  │ ...  │Obj N │      │
│  │Monster│Item │Effect│      │      │      │      │      │      │
│  └──┬───┴──┬───┴──┬───┴──────┴──────┴──────┴──────┴──────┘      │
│     │      │      │                                             │
└─────│──────│──────│─────────────────────────────────────────────┘
      │      │      │
      ▼      ▼      ▼
┌─────────────────────────────────────────────────────────────────┐
│  POLYGON OBJECT LISTS                                           │
│  Each polygon has a linked list of objects inside it            │
│                                                                 │
│  Polygon 0: Obj 0 → Obj 5 → Obj 12 → NULL                       │
│  Polygon 1: Obj 1 → NULL                                        │
│  Polygon 2: Obj 2 → Obj 7 → NULL                                │
│                                                                 │
│  Enables fast "what's in this room?" queries                    │
└─────────────────────────────────────────────────────────────────┘
```

### Entity Limits

```c
#define MAXIMUM_MONSTERS_PER_MAP 512
#define MAXIMUM_PROJECTILES_PER_MAP 128
#define MAXIMUM_OBJECTS_PER_MAP 512
#define MAXIMUM_EFFECTS_PER_MAP 64
```

---

## 8.3 Monster System

The monster system is the most complex entity system, handling 47 different monster types with AI behaviors.

### Monster Types and Factions

**Factions:**
- **Pfhor** - Fighters, Troopers, Hunters, Enforcers, Juggernauts
- **Compilers** - AI entities, teleport, invisible variants
- **Cyborgs** - Projectile, flamethrower
- **Humans** - Crew, scientists, security (allies)
- **Native** - Ticks, Yetis (water/sewage/lava variants)

### Monster Definition Structure

Each monster type is defined by a static structure:

```c
// From monster_definitions.h:147 (128 bytes)
struct monster_definition {
    short collection;                  // Sprite set
    short vitality;                    // Hit points

    unsigned long immunities;          // Damage types ignored
    unsigned long weaknesses;          // Amplified damage
    unsigned long flags;

    long monster_class, friends, enemies;  // Faction relationships

    world_distance radius, height;     // Collision bounds
    world_distance visual_range, dark_visual_range;
    short half_visual_arc, half_vertical_visual_arc;
    short intelligence;                // Pathfinding quality
    short speed, gravity, terminal_velocity;

    short attack_frequency;
    struct attack_definition melee_attack, ranged_attack;
};
```

### Monster AI State Machine

Monsters use a **two-layer state system**:

1. **Action State** - What the monster is physically doing (moving, attacking, dying)
2. **Mode** - Target lock status (has target, searching, no target)

This separation allows a monster to remain "locked" on a target even while performing different actions.

#### Action State Summary

| State | Description | Next States |
|-------|-------------|-------------|
| `_stationary` | Idle, no path | `_moving` |
| `_moving` | Walking toward target | `_attacking_close`, `_attacking_far`, `_stationary` |
| `_attacking_close` | Melee attack | `_waiting_to_attack_again` |
| `_attacking_far` | Ranged attack | `_waiting_to_attack_again` |
| `_waiting_to_attack_again` | Cooldown after attack | `_moving` |
| `_being_hit` | Stun from damage | `_moving` |
| `_dying_soft` | Collapsing death | removed |
| `_dying_hard` | Explosive death | removed |
| `_dying_flaming` | Fire death | removed |
| `_teleporting_in` | Fade-in (Compilers) | `_teleporting` |
| `_teleporting` | Active/invisible | `_teleporting_out` |
| `_teleporting_out` | Fade-out | `_stationary` |

#### Action State Flow Diagram

```
COMBAT FLOW (from monsters.h:129-142):

                            ┌───────────────┐
      ┌────────────────────►│  _stationary  │◄─────────────────────┐
      │                     └───────┬───────┘                      │
      │                             │                              │
      │                    See target / Path found                 │
      │                             ▼                              │
      │                     ┌───────────────┐         No path      │
      │    ┌───────────────►│   _moving     │─────────────────────►│
      │    │                └───────┬───────┘                      │
      │    │                        │                              │
      │    │              In attack range?                         │
      │    │            ┌───────────┴───────────┐                  │
      │    │    Melee range              Ranged distance           │
      │    │            ▼                       ▼                  │
      │    │   ┌─────────────────┐     ┌─────────────────┐         │
      │    │   │ _attacking_close│     │ _attacking_far  │         │
      │    │   └────────┬────────┘     └────────┬────────┘         │
      │    │            └───────────┬───────────┘                  │
      │    │                Attack complete                        │
      │    │                        ▼                              │
      │    │           ┌───────────────────────────┐               │
      │    └───────────│ _waiting_to_attack_again  │───────────────┘
      │                └───────────────────────────┘
      │                       Cooldown done
      │
 Hit by damage
      ▼
┌───────────────┐
│  _being_hit   │──────► (back to _moving after stun recovery)
└───────────────┘


DEATH STATES (triggered by lethal damage from any state):

┌─────────────────────┐     ┌────────────────────┐     ┌──────────────────────┐
│  _dying_soft        │     │  _dying_hard       │     │  _dying_flaming      │
│  (collapse anim)    │     │  (explode anim)    │     │  (fire death anim)   │
└──────────┬──────────┘     └─────────┬──────────┘     └──────────┬───────────┘
           │                          │                           │
           └──────────────────────────┼───────────────────────────┘
                                      ▼
                               Monster removed


TELEPORT STATES (Compiler monsters only):

┌──────────────────┐     ┌───────────────┐     ┌───────────────────┐
│ _teleporting_in  │────►│  _teleporting │────►│ _teleporting_out  │
│   (fade in)      │     │  (invisible)  │     │    (fade out)     │
└──────────────────┘     └───────────────┘     └───────────────────┘
```

### Monster Mode System (Target Lock)

Separate from action states, monsters have a **mode** that tracks target lock status:

```c
enum /* monster modes */
{
    _monster_locked,       // Has valid target in sight
    _monster_losing_lock,  // Target moved, searching
    _monster_lost_lock,    // Target out of sight
    _monster_unlocked,     // No target
    _monster_running       // Fleeing (civilians)
};
```

**Mode Transition Diagram:**

```
                              Target acquired
                                    │
                                    ▼
                        ┌───────────────────┐
          ┌────────────►│  _monster_locked  │◄─────────────┐
          │             └─────────┬─────────┘              │
          │                       │                        │
          │               Target moved to                  │
          │               another polygon                  │
          │                       │                        │
    Target visible               ▼                    Target visible
    again              ┌─────────────────────┐        again
          │            │ _monster_losing_lock│             │
          │            └─────────┬───────────┘             │
          │                      │                         │
          │              Changed polygons                  │
          │              > intelligence times              │
          │                      │                         │
          │                      ▼                         │
          │            ┌─────────────────────┐             │
          └────────────│  _monster_lost_lock │─────────────┘
                       └─────────┬───────────┘
                                 │
                         No target found
                         after search
                                 │
                                 ▼
                       ┌─────────────────────┐
                       │  _monster_unlocked  │
                       └─────────────────────┘

Intelligence determines lock persistence:
  _intelligence_low = 2     polygon changes before losing lock
  _intelligence_average = 3 polygon changes before losing lock
  _intelligence_high = 8    polygon changes before losing lock
```

### Monster Classes and Factions

Monsters use a bitmask system for friend/enemy relationships:

**Class Bitmask System:**

| Class | Bit | Monster Type IDs | Description |
|-------|-----|------------------|-------------|
| `_class_player` | 0 | - | Player marine (not a monster type) |
| `_class_human_civilian` | 1 | 14-16 | Crew (green), Scientist (blue), Security (red BOBs) |
| `_class_madd` | 2 | 17 | Rampant BOB (hostile human) |
| `_class_possessed_hummer` | 3 | 42 | Durandal's controlled hummer |
| `_class_defender` | 4 | 21-22 | Minor/Major Defender |
| `_class_fighter` | 5 | 0-3 | Minor/Major/Minor Proj/Major Proj Fighters |
| `_class_trooper` | 6 | 6-7 | Minor/Major Troopers |
| `_class_hunter` | 7 | 8-9 | Minor/Major Hunters |
| `_class_enforcer` | 8 | 10-11 | Minor/Major Enforcers |
| `_class_juggernaut` | 9 | 23-24 | Minor/Major Juggernauts |
| `_class_hummer` | 10 | 12-13 | Minor/Major Hummers |
| `_class_compiler` | 11 | 4-5 | Minor/Major Compilers (S'pht) |
| `_class_cyborg` | 12 | 18-19 | Minor/Major Cyborgs |
| `_class_assimilated_civilian` | 13 | 20 | Assimilated BOB (explodes) |
| `_class_tick` | 14 | 25-27 | Tick/Boom Tick/Kamikaze Tick |
| `_class_yeti` | 15 | 28-30 | Yeti/Sewage Yeti/Lava Yeti |

**Faction Groupings with All Members:**

| Faction | Bitmask | Member Classes | Monster Types |
|---------|---------|----------------|---------------|
| **Human** | `0x000F` | player, civilian, madd, possessed_hummer | BOBs, Security, Rampant |
| **Pfhor** | `0x03E0` | fighter, trooper, hunter, enforcer, juggernaut | All Pfhor soldiers |
| **Client** | `0x3C00` | compiler, cyborg, assimilated_civilian, hummer | S'pht, Cyborgs |
| **Native** | `0xC000` | tick, yeti | Environmental creatures |
| **Defender** | `0x0010` | defender | Floating drones |

```c
// Faction bitmask definitions (from monster_definitions.h)
#define _class_human    (_class_player | _class_human_civilian | _class_madd | _class_possessed_hummer)
#define _class_pfhor    (_class_fighter | _class_trooper | _class_hunter | _class_enforcer | _class_juggernaut)
#define _class_client   (_class_compiler | _class_assimilated_civilian | _class_cyborg | _class_hummer)
#define _class_native   (_class_tick | _class_yeti)
```

**Default Faction Relationships:**

| Faction | Friends | Enemies |
|---------|---------|---------|
| Human | Human | Pfhor, Client, Native |
| Pfhor | Pfhor, Client | Human, Native |
| Client | Pfhor, Client | Human |
| Native | Native | Human, Pfhor |
| Defender | Pfhor | Human |

**Attitude Calculation:**

```c
short get_monster_attitude(short monster_index, short target_index) {
    if (definition->friends & target_class) return _friendly;
    if (definition->enemies & target_class) return _hostile;
    return _neutral;
}
```

### Monster Flags Reference

| Flag | Value | Description |
|------|-------|-------------|
| `_monster_is_omniscent` | 0x1 | Ignores line-of-sight for targeting |
| `_monster_flys` | 0x2 | Can move vertically freely |
| `_monster_is_alien` | 0x4 | Slower on easier difficulties |
| `_monster_major` | 0x8 | Type-1 is minor variant |
| `_monster_minor` | 0x10 | Type+1 is major variant |
| `_monster_cannot_be_dropped` | 0x20 | Always spawns regardless of difficulty |
| `_monster_floats` | 0x40 | Gradual vertical movement |
| `_monster_cannot_attack` | 0x80 | Runs to safety (civilians) |
| `_monster_uses_sniper_ledges` | 0x100 | Positions on elevated platforms |
| `_monster_is_invisible` | 0x200 | Uses invisibility transfer mode |
| `_monster_is_subtly_invisible` | 0x400 | Partial invisibility |
| `_monster_is_kamakazi` | 0x800 | Suicides when close to target |
| `_monster_is_berserker` | 0x1000 | Goes berserk below 1/4 vitality |
| `_monster_is_enlarged` | 0x2000 | 1.25× normal height |
| `_monster_has_delayed_hard_death` | 0x4000 | Soft death then hard death |
| `_monster_fires_symmetrically` | 0x8000 | Fires at ±dy simultaneously |
| `_monster_has_nuclear_hard_death` | 0x10000 | Screen flash on death |
| `_monster_cant_fire_backwards` | 0x20000 | Max 135° turn to fire |
| `_monster_can_die_in_flames` | 0x40000 | Uses flaming death animation |
| `_monster_waits_with_clear_shot` | 0x80000 | Holds position if has clear shot |
| `_monster_is_tiny` | 0x100000 | 0.25× normal height |
| `_monster_attacks_immediately` | 0x200000 | No delay before first attack |

### Monster Speed Table

| Speed Constant | Value | Units/Second |
|----------------|-------|--------------|
| `_speed_slow` | WORLD_ONE/120 | ~8.5 |
| `_speed_medium` | WORLD_ONE/80 | ~12.8 |
| `_speed_almost_fast` | WORLD_ONE/70 | ~14.6 |
| `_speed_fast` | WORLD_ONE/40 | ~25.6 |
| `_speed_superfast1` | WORLD_ONE/30 | ~34.1 |
| `_speed_blinding` | WORLD_ONE/20 | ~51.2 |
| `_speed_insane` | WORLD_ONE/10 | ~102.4 |

---

## 8.4 AI Load Distribution

Marathon distributes expensive AI operations across frames to maintain performance.

### The Problem

```
Naive approach (all AI every frame):

Frame 1: 50 monsters × pathfinding = 50 expensive operations!
         Game runs at 5 FPS...

Frame 2: 50 monsters × pathfinding = 50 expensive operations!
         Still 5 FPS...
```

### Marathon's Solution

```
AI Load Distribution:

Frame N:    Monster 0 gets target search time
Frame N+1:  Monster 1 gets target search time
Frame N+2:  Monster 2 gets target search time
...

Frame N:    Monster 0 gets pathfinding time
Frame N+4:  Monster 1 gets pathfinding time  (pathfinding is 1 per 4 frames)
Frame N+8:  Monster 2 gets pathfinding time
...
```

**Key Variables:**

```c
dynamic_world->last_monster_index_to_get_time   // Round-robin for targeting
dynamic_world->last_monster_index_to_build_path // Round-robin for pathfinding
```

**Rules:**
- Only ONE monster gets expensive target search per frame
- Only ONE monster gets pathfinding per 4 frames
- Monsters without paths get immediate pathfinding regardless
- When all monsters have had a turn, index resets to -1

---

## 8.5 Activation System

Monsters start inactive and must be triggered to engage.

### Activation Ranges

```c
GLUE_TRIGGER_ACTIVATION_RANGE = 8 * WORLD_ONE   // Trigger activation
MONSTER_ALERT_ACTIVATION_RANGE = 5 * WORLD_ONE  // Sound/sight activation
```

### Activation Biases (Set in Editor)

```c
_activate_on_player           // Target player immediately
_activate_on_nearest_hostile  // Find closest enemy
_activate_on_goal             // Move toward goal polygon
_activate_randomly            // Random behavior
```

### Activation Flags

```c
_pass_one_zone_border            // Can cross one zone
_passed_zone_border              // Has crossed a zone
_activate_invisible_monsters     // Sound/teleport trigger
_activate_deaf_monsters          // Trigger (not sound)
_pass_solid_lines                // Trigger (not sound)
_use_activation_biases           // Follow editor instructions
_activation_cannot_be_avoided    // Cannot be suppressed
```

### Activation Flood Algorithm

```
1. Start flood from caller's polygon
2. For each polygon reached:
   - Check all objects in polygon
   - If object is monster and meets criteria:
     - Activate if inactive
     - Lock on target if hostile
     - Propagate activation to neighbors
3. Continue until maximum cost or no more polygons

Flood respects:
  - Zone boundaries (unless explicitly crossing)
  - Line-of-sight for sound activation
  - Deaf/blind monster flags
```

---

## 8.6 Pathfinding System

Marathon's pathfinding uses a two-layer system: a flood fill algorithm for exploring polygon connectivity, and a path builder that extracts waypoints for navigation.

### System Limits

```c
#define MAXIMUM_PATHS 20              // Concurrent paths (pathfinding.c)
#define MAXIMUM_POINTS_PER_PATH 63    // Waypoints per path
#define MAXIMUM_FLOOD_NODES 255       // Search nodes (flood_map.c)
```

### Data Structures

**Flood Fill Node** [flood_map.c:39]:

```c
struct node_data {           // 16 bytes
    word flags;              // NODE_IS_EXPANDED bit flag
    short parent_node_index; // For backtracking
    short polygon_index;     // This polygon
    long cost;               // Accumulated cost to reach here
    short depth;             // Polygons from start
    long user_flags;         // Caller-defined flags
};
```

**Path Definition** (pathfinding.c):

```c
struct path_definition {     // 256 bytes
    short current_step;      // Current waypoint index
    short step_count;        // Total waypoints (NONE = free)
    world_point2d points[MAXIMUM_POINTS_PER_PATH];  // Waypoint coordinates
};
```

### Flood Fill Modes

The flood map supports four search strategies:

| Mode | Description | Use Case |
|------|-------------|----------|
| `_best_first` | Expands lowest-cost node first | Optimal path, slower |
| `_breadth_first` | Expands in order added | Faster for large areas |
| `_flagged_breadth_first` | Breadth-first with user flags | Special constraints |
| `_depth_first` | Deepest node first | Not implemented |

```
_best_first (A* style):          _breadth_first (BFS):

  Expand lowest cost →             Expand in order added →
  ┌───┐                            ┌───┐
  │ 3 │ ← Skip                     │ 1 │ ← Expand first
  ├───┤                            ├───┤
  │ 1 │ ← Expand!                  │ 2 │ ← Expand second
  ├───┤                            ├───┤
  │ 5 │ ← Skip                     │ 3 │ ← Expand third
  └───┘                            └───┘
```

### Cost Function

```c
typedef long (*cost_proc_ptr)(
    short source_polygon_index,
    short line_index,
    short destination_polygon_index,
    void *caller_data
);

// Monster pathfinding costs:
MONSTER_PATHFINDING_OBSTRUCTION_COST = 2 * WORLD_ONE²  // Objects blocking
MONSTER_PATHFINDING_PLATFORM_COST = 4 * WORLD_ONE²     // Moving platforms
MINIMUM_MONSTER_PATHFINDING_POLYGON_AREA = WORLD_ONE   // Polygon too small
```

**Cost function returns:**
- Positive value: Add to path cost
- Zero or negative: Polygon not traversable (blocked)
- NULL cost_proc: Use polygon area as cost (fastest)

### Path Creation Flow

```
new_path(source, destination, cost_func)
                │
                ▼
    ┌──────────────────────────────────────┐
    │ flood_map(source_polygon, ...)       │
    │ Start flood fill from source         │
    └───────────────────┬──────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────┐
    │ while (polygon != destination)       │
    │   flood_map(NONE, ...)               │
    │   Expand next lowest-cost node       │
    └───────────────────┬──────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────┐
    │ reverse_flood_map()                  │
    │ Backtrack via parent_node_index      │
    │ Build path from destination to source│
    └───────────────────┬──────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────┐
    │ calculate_midpoint_of_shared_line()  │
    │ For each polygon transition:         │
    │ Generate waypoint on shared edge     │
    └──────────────────────────────────────┘
```

### Waypoint Generation

Waypoints are placed on the shared edge between adjacent polygons:

```
  ┌─────────────────┐
  │  Polygon A      │
  │       ●───────────────────► Monster path
  │       │         │
  └───────┼─────────┘
          │ ← Waypoint on shared line
  ┌───────┼─────────┐
  │       │         │
  │       ●         │
  │  Polygon B      │
  └─────────────────┘
```

The `calculate_midpoint_of_shared_line()` function finds the midpoint of the shared edge, respecting a minimum separation from walls.

### Path Following

```c
// Get next waypoint
boolean move_along_path(short path_index, world_point2d *p) {
    if (current_step < step_count) {
        *p = points[current_step++];
        return TRUE;
    }
    return FALSE;  // Path complete
}
```

**Path Invalidation Triggers:**
- Target moves more than 2 WORLD_ONE
- Monster completes attack
- Monster takes damage (stun recovery)
- Path blocked by closed door

---

## 8.7 Combat System

### Attack Definition Structure

```c
struct attack_definition {
    short type;                    // Projectile type
    short repetitions;             // Shots per attack
    angle error;                   // ± accuracy spread
    world_distance range;          // Maximum attack range
    short attack_shape;            // Animation keyframe for attack
    world_distance dx, dy, dz;     // Projectile spawn offset
};
```

### Attack Timing

```
Animation starts
    │
    ▼ (frames pass)
    │
Keyframe reached ──────► Projectile spawned / Damage dealt
    │
    ▼ (more frames)
    │
Animation ends ──────► Cooldown begins
    │
    ▼ (cooldown ticks)
    │
Ready to attack again
```

### Attack Frequency

- Stationary monsters: 2× attacks/second typical
- Moving monsters: Half attack rate
- Varies by monster type (1-4 seconds between attacks)

### Melee vs Ranged Selection

- If within melee range and has melee attack → melee
- If within ranged range and has ranged attack → ranged
- `_monster_chooses_weapons_randomly` flag randomizes selection

---

## 8.8 Weapon System

Marathon features 10 player weapons with complex state machines.

### Weapon Classes

```c
_melee_class              // Fist
_normal_class             // Single trigger, one ammo type
_dual_function_class      // Primary/secondary different
_twofisted_pistol_class   // Dual wield
_multipurpose_class       // Rifle + grenade launcher
```

### Weapon Types (weapons.h:9-25)

| ID | Enum Constant | Weapon | Ammo/Mag | Notes |
|----|---------------|--------|----------|-------|
| 0 | `_weapon_fist` | Fist | ∞ | Melee only |
| 1 | `_weapon_pistol` | Magnum Pistol | 8 | Can dual-wield |
| 2 | `_weapon_plasma_pistol` | Fusion Pistol | 20 | Chargeable |
| 3 | `_weapon_assault_rifle` | MA-75B Rifle | 52 | +7 grenades secondary |
| 4 | `_weapon_missile_launcher` | SPNKR-X17 | 2 | Guided missiles |
| 5 | `_weapon_flamethrower` | TOZT-7 | 210 ticks | Continuous flame |
| 6 | `_weapon_alien_shotgun` | Alien Weapon | Varies | Dropped by enemies |
| 7 | `_weapon_shotgun` | WSTE-M5 | 2 | Can dual-wield |
| 8 | `_weapon_ball` | Ball | - | Multiplayer only |
| 9 | `_weapon_smg` | KKV-7 SMG | 32 | Burst fire |

```c
// From weapons.h
enum { /* Weapons */
    _weapon_fist,                    // 0
    _weapon_pistol,                  // 1
    _weapon_plasma_pistol,           // 2
    _weapon_assault_rifle,           // 3
    _weapon_missile_launcher,        // 4
    _weapon_flamethrower,            // 5
    _weapon_alien_shotgun,           // 6
    _weapon_shotgun,                 // 7
    _weapon_ball,                    // 8
    _weapon_smg,                     // 9
    MAXIMUM_NUMBER_OF_WEAPONS,       // 10

    _weapon_doublefisted_pistols = MAXIMUM_NUMBER_OF_WEAPONS,  // 10 (pseudo)
    _weapon_doublefisted_shotguns,                              // 11 (pseudo)
    PLAYER_TORSO_SHAPE_COUNT                                    // 12
};
```

### Weapon Definition Structure

```c
// From weapon_definitions.h:169 (196 bytes)
struct weapon_definition {
    short weapon_class, flags;

    fixed idle_height, bob_amplitude, kick_height, reload_height;

    short collection;
    short idle_shape, firing_shape, reloading_shape;

    short ready_ticks, await_reload_ticks, loading_ticks;

    struct trigger_definition weapons_by_trigger[2];  // Primary/secondary
};

struct trigger_definition {
    short rounds_per_magazine;
    short ammunition_type;
    short ticks_per_round;         // Fire rate
    short recovery_ticks;
    short projectile_type;
    short theta_error;             // Spread angle
    short burst_count;             // Pellets per shot
};
```

### Weapon State Machine

```c
_weapon_idle, _weapon_raising, _weapon_lowering
_weapon_charging, _weapon_charged
_weapon_firing, _weapon_recovering
_weapon_awaiting_reload, _weapon_waiting_to_load
_weapon_finishing_reload
```

**Weapon State Machine Diagram:**

```
                              ┌──────────────────────────────────────────────────────┐
                              │                                                      │
                              ▼                                                      │
                      ┌───────────────┐                                              │
                      │  _weapon_idle │◄──────────────────────────────────┐          │
                      └───────┬───────┘                                   │          │
                              │                                           │          │
          ┌───────────────────┼───────────────────┐                       │          │
          │                   │                   │                       │          │
    Switch weapon      Fire pressed        Need reload                    │          │
          │                   │                   │                       │          │
          ▼                   ▼                   ▼                       │          │
  ┌───────────────┐   ┌───────────────┐   ┌─────────────────┐             │          │
  │   _lowering   │   │   _charging   │   │ _awaiting_reload│             │          │
  └───────┬───────┘   └───────┬───────┘   └────────┬────────┘             │          │
          │                   │                    │                      │          │
          │           Charge complete       Start loading                 │          │
          │                   │                    │                      │          │
          │                   ▼                    ▼                      │          │
          │           ┌───────────────┐   ┌─────────────────┐             │          │
          │           │   _charged    │   │ _waiting_to_load│             │          │
          │           └───────┬───────┘   └────────┬────────┘             │          │
          │                   │                    │                      │          │
          │           Release or                 Load ammo               │          │
          │           Overload                     │                      │          │
          │                   │                    ▼                      │          │
          │                   │           ┌─────────────────┐             │          │
          │                   │           │_finishing_reload│─────────────┘          │
          │                   │           └─────────────────┘                        │
          │                   │                                                      │
          │                   ▼                                                      │
          │           ┌───────────────┐                                              │
          │           │   _firing     │──────────────┐                               │
          │           └───────────────┘              │                               │
          │                                    Animation done                        │
          │                                          │                               │
          │                                          ▼                               │
          │                                  ┌───────────────┐                       │
          │                                  │  _recovering  │───────────────────────┘
          │                                  └───────────────┘
          │
          ▼
  ┌───────────────┐
  │   _raising    │  (when new weapon selected)
  └───────┬───────┘
          │
          └──────────────────────────────────────────────────────────────────────────┘

Special states for dual-wielded weapons (pistols, shotguns):
  _lowering_for_twofisted_reload    - One hand lowers so other can reload
  _awaiting_twofisted_reload        - Waiting for other hand to lower
  _waiting_for_twofist_to_reload    - Offscreen, waiting for partner
  _sliding_over_to_second_position  - Pistol moving to akimbo position
  _sliding_over_from_second_position - Returning to center
  _waiting_for_other_idle_to_reload  - Waiting for partner to be idle
```

### Firing Pipeline

1. Input check (trigger pressed)
2. Ammo validation
3. Rate of fire check (ticks_per_round)
4. Projectile spawn via `new_projectile()`
5. Effects (shell casings, sounds, light)
6. Recovery wait
7. Reload after magazine empty

---

## 8.9 Shell Casing System

Shell casings are first-person visual effects that spawn when certain weapons fire.

### Shell Casing Types (weapon_definitions.h:82-91)

| ID | Enum Constant | Description | Used By |
|----|---------------|-------------|---------|
| 0 | `_shell_casing_assault_rifle` | MA-75B brass | Assault Rifle |
| 1 | `_shell_casing_pistol` | Magnum brass (center) | Pistol (single) |
| 2 | `_shell_casing_pistol_left` | Magnum brass (left hand) | Dual Pistols |
| 3 | `_shell_casing_pistol_right` | Magnum brass (right hand) | Dual Pistols |
| 4 | `_shell_casing_smg` | SMG brass | KKV-7 SMG |

### Data Structures

```c
// From weapon_definitions.h:93-100
struct shell_casing_definition {
    short collection, shape;     // Graphics source (_collection_weapons_in_hand)
    fixed x0, y0;               // Initial position (fixed-point, 0-FIXED_ONE screen coords)
    fixed vx0, vy0;             // Initial velocity (fixed-point units per tick)
    fixed dvx, dvy;             // Velocity delta per tick (gravity/friction)
};

// From weapons.c:120-129 - Runtime state per casing
struct shell_casing_data {
    short type;                 // Shell casing type enum
    short frame;                // Current animation frame
    word flags;                 // _shell_casing_is_reversed for left-hand
    fixed x, y;                 // Current screen position
    fixed vx, vy;               // Current velocity
};
```

### How Shell Casings Work

**1. Spawning** (`new_shell_casing()`, weapons.c:3686):
- Called when weapon fires (from fire_weapon trigger handlers)
- Finds free slot in player's shell_casings array (max 4)
- Copies initial position/velocity from shell_casing_definition
- Adds randomization: random starting frame, position jitter
- If reversed flag set (left-hand weapon), negates X velocity

**2. Physics Update** (`update_shell_casings()`, weapons.c:3722):
- Called every tick from `update_player_weapons()`
- For each active casing:
  ```c
  shell_casing->x += shell_casing->vx;  // Move horizontally
  shell_casing->y += shell_casing->vy;  // Move vertically
  shell_casing->vx += definition->dvx;  // Apply horizontal drag
  shell_casing->vy += definition->dvy;  // Apply gravity (negative = up!)
  ```
- Removal condition: `x >= FIXED_ONE` or `x < 0` (off screen sides)

**3. Rendering** (`get_shell_casing_display_data()`, weapons.c:3749):
- Called by weapon rendering system when building display list
- Returns weapon_display_information for each active casing
- Advances animation frame each call
- Position converted: `vertical_position = FIXED_ONE - y` (screen Y is inverted)

### Key Constants

- `MAXIMUM_SHELL_CASINGS = 4` - Max active per player at once
- `_shell_casing_is_reversed = 0x0001` - Flip X velocity for left hand
- Position coordinates: 0 = left/top, FIXED_ONE = right/bottom

### Which Weapons Use Shell Casings

- **Assault Rifle:** Yes (type 0)
- **Pistol:** Yes (types 1-3 depending on dual-wield state)
- **SMG:** Yes (type 4)
- **Shotgun:** No (shells eject during reload animation instead)
- **Fusion Pistol:** No (energy weapon)
- **Flamethrower:** No (continuous stream)
- **Alien Weapon:** No (energy weapon)
- **Rocket Launcher:** No (no casings)

---

## 8.10 Projectile System

Marathon features 39 projectile types covering all weapons and monsters.

### Projectile Types (projectiles.h:13-53)

| ID | Enum Constant | Description | Source |
|----|---------------|-------------|--------|
| 0 | `_projectile_rocket` | SPNKR rocket | Player |
| 1 | `_projectile_grenade` | MA-75B grenade | Player |
| 2 | `_projectile_pistol_bullet` | Magnum round | Player |
| 3 | `_projectile_rifle_bullet` | MA-75B round | Player |
| 4 | `_projectile_shotgun_bullet` | Shotgun pellet | Player |
| 5 | `_projectile_staff` | Staff melee | Pfhor |
| 6 | `_projectile_staff_bolt` | Staff ranged | Pfhor |
| 7 | `_projectile_flamethrower_burst` | Flame | Player |
| 8 | `_projectile_compiler_bolt_minor` | Minor compiler | Compiler |
| 9 | `_projectile_compiler_bolt_major` | Major compiler | Compiler |
| 10 | `_projectile_alien_weapon` | Alien shotgun | Player/Pfhor |
| 11 | `_projectile_fusion_bolt_minor` | Fusion tap | Player |
| 12 | `_projectile_fusion_bolt_major` | Fusion charged | Player |
| 13 | `_projectile_hunter` | Hunter bolt | Hunter |
| 14 | `_projectile_fist` | Melee punch | Player |
| 15 | `_projectile_unused` | (Reserved) | - |
| 16 | `_projectile_armageddon_electricity` | Special | Cyborg |
| 17 | `_projectile_juggernaut_rocket` | Juggernaut missile | Juggernaut |
| 18 | `_projectile_trooper_bullet` | Trooper rifle | Trooper |
| 19 | `_projectile_trooper_grenade` | Trooper grenade | Trooper |
| 20 | `_projectile_minor_defender` | Defender attack | Defender |
| 21 | `_projectile_major_defender` | Defender attack | Defender |
| 22 | `_projectile_juggernaut_missile` | Juggernaut rocket | Juggernaut |
| 23 | `_projectile_minor_energy_drain` | S'pht drain | S'pht |
| 24 | `_projectile_major_energy_drain` | S'pht drain | S'pht |
| 25 | `_projectile_oxygen_drain` | Oxygen damage | Environment |
| 26 | `_projectile_minor_hummer` | Hummer attack | Hummer |
| 27 | `_projectile_major_hummer` | Hummer attack | Hummer |
| 28 | `_projectile_durandal_hummer` | Special hummer | Hummer |
| 29 | `_projectile_minor_cyborg_ball` | Cyborg ball | Cyborg |
| 30 | `_projectile_major_cyborg_ball` | Cyborg ball | Cyborg |
| 31 | `_projectile_ball` | Game ball | Multiplayer |
| 32 | `_projectile_minor_fusion_dispersal` | Fusion shrapnel | Effect |
| 33 | `_projectile_major_fusion_dispersal` | Fusion shrapnel | Effect |
| 34 | `_projectile_overloaded_fusion_dispersal` | Fusion explosion | Effect |
| 35 | `_projectile_yeti` | Yeti attack | Yeti |
| 36 | `_projectile_sewage_yeti` | Sewage yeti | Sewage Yeti |
| 37 | `_projectile_lava_yeti` | Lava yeti | Lava Yeti |
| 38 | `_projectile_smg_bullet` | SMG round | Player |

### Projectile Definition Structure

```c
// From projectile_definitions.h:36 (54 bytes)
struct projectile_definition {
    short collection, shape;
    short detonation_effect, media_detonation_effect;
    short contrail_effect;
    short ticks_between_contrails;

    world_distance radius;         // Hit radius
    world_distance area_of_effect; // Damage radius
    struct damage_definition damage;

    unsigned long flags;

    world_distance speed;
    world_distance maximum_range;
};
```

### Projectile Flags

- `_guided` - Homing
- `_affected_by_gravity` - Falls
- `_persistent` - Doesn't vanish
- `_rebounds_from_floor` - Bounces
- `_penetrates_media` - Water pass-through
- `_horizontal_wander` / `_vertical_wander` - Spread

### Damage Types (from map.h)

| Type | ID | Source | Notes |
|------|-----|--------|-------|
| `_damage_explosion` | 0 | Rockets, grenades | Area effect |
| `_damage_electrical_staff` | 1 | Staff weapon | Alien |
| `_damage_projectile` | 2 | Bullets | Standard |
| `_damage_absorbed` | 3 | Shield absorbed | No effect |
| `_damage_flame` | 4 | Flamethrower | Continuous |
| `_damage_hound_claws` | 5 | Hound attack | Melee |
| `_damage_alien_projectile` | 6 | Alien shots | Various |
| `_damage_hulk_slap` | 7 | Hulk attack | Melee |
| `_damage_compiler_bolt` | 8 | Compiler attack | Energy |
| `_damage_fusion_bolt` | 9 | Fusion pistol | Energy |
| `_damage_hunter_bolt` | 10 | Hunter shot | Energy |
| `_damage_fist` | 11 | Punch | Melee |
| `_damage_teleporter` | 12 | Telefrag | Instant kill |
| `_damage_defender` | 13 | Defender attack | Energy |
| `_damage_yeti_claws` | 14 | Yeti attack | Melee |
| `_damage_yeti_projectile` | 15 | Yeti shot | Projectile |
| `_damage_crushing` | 16 | Platform/door | Environmental |
| `_damage_lava` | 17 | Lava contact | Environmental |
| `_damage_suffocation` | 18 | No oxygen | Environmental |
| `_damage_goo` | 19 | Sewage/goo | Environmental |
| `_damage_energy_drain` | 20 | Shield drain | Special |
| `_damage_oxygen_drain` | 21 | O2 drain | Special |
| `_damage_hummer_bolt` | 22 | Hummer attack | Energy |
| `_damage_shotgun_projectile` | 23 | Shotgun | Multi-hit |

**Monster Immunities/Weaknesses:** Stored as bitmasks using `FLAG(_damage_type)`

---

## 8.11 Effects System

Marathon uses 85+ effect types for visual feedback.

### Effect Categories

- **Impact:** Bullet ricochet, blood, sparks
- **Environmental:** Water/lava splashes (3 sizes each)
- **Weapon:** Rocket contrails, shell casings
- **Teleport:** In/out effects
- **Death:** Faction-specific

### Effect Data Structure

```c
// From effects.h:90 (16 bytes)
struct effect_data {
    short type;
    short object_index;    // Sprite
    word flags;
    short data;            // Special data
    short delay;           // Visibility delay
};
```

### Effect Lifecycle

1. Spawn via `new_effect()`
2. Delay (invisible ticks)
3. Animation play
4. Sound trigger
5. Auto-removal after animation

**Maximum:** 64 simultaneous effects.

---

## 8.12 Motion Sensor System

The motion sensor (radar) displays nearby entities on a circular HUD element.

### Core Constants

```c
#define MAXIMUM_MOTION_SENSOR_ENTITIES 12
#define NUMBER_OF_PREVIOUS_LOCATIONS 6      // Trail effect
#define MOTION_SENSOR_UPDATE_FREQUENCY 5    // Ticks between updates
#define MOTION_SENSOR_RESCAN_FREQUENCY 15   // Ticks between full rescans
#define MOTION_SENSOR_RANGE (8*WORLD_ONE)   // Detection radius
#define MOTION_SENSOR_SCALE 7               // World-to-screen scale
#define FLICKER_FREQUENCY 0xf               // Magnetic interference
```

### Entity Data Structure

```c
struct entity_data {
    word flags;                 // [slot_used.1] [being_removed.1] [unused.14]
    short monster_index;
    shape_descriptor shape;     // Blip appearance
    short remove_delay;         // Fade-out counter [0, NUMBER_OF_PREVIOUS_LOCATIONS)
    point2d previous_points[NUMBER_OF_PREVIOUS_LOCATIONS];  // Trail history
    boolean visible_flags[NUMBER_OF_PREVIOUS_LOCATIONS];
    world_point3d last_location;
    angle last_facing;
};
```

### Motion Sensor Pipeline

```
motion_sensor_scan() called every tick:
      │
      ├─► ticks_since_last_update < UPDATE_FREQUENCY?
      │           │
      │           └─► YES: Return (no update)
      │
      ├─► Reset update counter
      │
      ├─► ticks_since_last_rescan >= RESCAN_FREQUENCY?
      │           │
      │           └─► YES: Scan for new entities in range
      │                   │
      │                   ├─► For each monster in range:
      │                   │     └─► find_or_add_motion_sensor_entity()
      │                   │
      │                   └─► Mark out-of-range entities for removal
      │
      ├─► Update entity positions
      │     └─► Store in previous_points[] ring buffer
      │
      └─► Set motion_sensor_changed = TRUE
```

### Blip Types

| Shape | Source | Meaning |
|-------|--------|---------|
| `alien_shapes` | Hostile aliens | Danger |
| `friendly_shapes` | Allied units | Friendly |
| `enemy_shapes` | Hostile players | Enemy (multiplayer) |
| `compass_shapes` | Network game | Team compass |

### Trail Effect

```
Blip movement shown as fading trail:

    Current position (brightest)
              ●
             ◐
            ◔
           ◌         Previous positions (fading)
          ○
         ·           Oldest position (faintest)

NUMBER_OF_PREVIOUS_LOCATIONS = 6 positions tracked
```

**Removal Animation:**
When entity leaves range, it's marked `SLOT_IS_BEING_REMOVED` and fades out over `remove_delay` ticks before slot is freed.

---

## 8.13 Terminal System

Terminals provide story content through an interactive text/image system.

### Terminal Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `#LOGON` | `#LOGON XXXX` | Login screen (XXXX = shape) |
| `#UNFINISHED` | `#UNFINISHED` | Unfinished mission text |
| `#SUCCESS` | `#SUCCESS` | Success mission text |
| `#FAILURE` | `#FAILURE` | Failure mission text |
| `#INFORMATION` | `#INFORMATION` | General information |
| `#CHECKPOINT` | `#CHECKPOINT XX` | Goal checkpoint (XX = goal) |
| `#SOUND` | `#SOUND XXXX` | Play sound effect |
| `#MOVIE` | `#MOVIE XXXX` | Play movie (from Movie file) |
| `#TRACK` | `#TRACK XXXX` | Play music (from Music file) |
| `#PICT` | `#PICT XXXX` | Display PICT image |
| `#INTERLEVEL TELEPORT` | `#INTERLEVEL TELEPORT XXX` | Go to level XXX |
| `#INTRALEVEL TELEPORT` | `#INTRALEVEL TELEPORT XXX` | Go to polygon XXX |
| `#END` | `#END` | End current group |

### Text Formatting Codes

| Code | Effect |
|------|--------|
| `$B` | Bold on |
| `$b` | Bold off |
| `$I` | Italic on |
| `$i` | Italic off |
| `$U` | Underline on |
| `$u` | Underline off |

### Terminal State Machine

```
Player approaches terminal
           │
           ▼
    ┌─────────────┐
    │ enter_computer│
    │ _interface() │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐     Tab/Space      ┌─────────────┐
    │ Display page ├──────────────────►│ Next page   │
    └──────┬──────┘                    └──────┬──────┘
           │                                  │
           │ Last page                        │
           ▼                                  │
    ┌─────────────┐◄──────────────────────────┘
    │ Check ending│
    │   command   │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │             │
Teleport?     Exit
    │             │
    ▼             ▼
goto_level() abort_terminal_mode()
```

---

## 8.14 Replay/Recording System

Marathon supports recording and playback of gameplay through action queue capture.

### Core Constants

```c
#define RECORD_CHUNK_SIZE (MAXIMUM_QUEUE_SIZE/2)
#define MAXIMUM_TIME_DIFFERENCE 15        // Ticks tolerance
#define MAXIMUM_NET_QUEUE_SIZE 8
#define MAXIMUM_REPLAY_SPEED 5
#define MINIMUM_REPLAY_SPEED (-5)
```

### Action Queue Structure

```c
struct action_queue {
    short read_index, write_index;
    long* buffer;               // Circular buffer of action flags
};
```

### Recording Process

```
Game Loop (recording):
       │
       ├─► Collect local input → action_flags
       │
       ├─► Store in action_queue
       │
       ├─► Every RECORD_CHUNK_SIZE actions:
       │     └─► Write chunk to replay file
       │
       └─► Continue until game ends
```

### Playback Modes

| Speed | Effect |
|-------|--------|
| `MINIMUM_REPLAY_SPEED (-5)` | Slowest playback |
| `-1` | Half speed |
| `0` | Normal speed (1x) |
| `1` | 2x speed |
| `MAXIMUM_REPLAY_SPEED (5)` | Fastest playback |
| `toggle_ludicrous_speed()` | Skip rendering (fastest) |

---

## 8.15 Save/Load System

Games are saved as Marathon WAD files containing complete world state.

### Save Game Tags

| Tag | Contents | Loaded from level? |
|-----|----------|-------------------|
| `ENDPOINT_DATA_TAG` | Vertex data | Yes |
| `LINE_TAG` | Line definitions | Yes |
| `SIDE_TAG` | Wall textures | Yes |
| `POLYGON_TAG` | Room data | Yes |
| `LIGHTSOURCE_TAG` | Light states | No |
| `OBJECT_TAG` | Map objects | Yes |
| `MAP_INFO_TAG` | Level metadata | Yes |
| `MEDIA_TAG` | Liquid states | No |
| `PLAYER_STRUCTURE_TAG` | Player data | No |
| `DYNAMIC_STRUCTURE_TAG` | World state | No |
| `OBJECT_STRUCTURE_TAG` | Object instances | No |
| `MONSTERS_STRUCTURE_TAG` | Monster states | No |
| `EFFECTS_STRUCTURE_TAG` | Active effects | No |
| `PROJECTILES_STRUCTURE_TAG` | Active projectiles | No |
| `PLATFORM_STRUCTURE_TAG` | Platform states | No |
| `WEAPON_STATE_TAG` | Weapon states | No |
| `TERMINAL_STATE_TAG` | Terminal progress | No |
| `AUTOMAP_LINES` | Explored lines | No |
| `AUTOMAP_POLYGONS` | Explored areas | No |

**Level vs Save Difference:**
- **Level load:** Only loads "loaded_by_level = TRUE" tags
- **Save load:** Loads ALL tags (complete state restoration)

---

## 8.16 See Also

- **[Chapter 6: Physics and Collision](06_physics.md)** - Movement physics used by monsters
- **[Chapter 7: Game Loop](07_game_loop.md)** - `update_monsters()` called from main loop
- **[Chapter 11: Performance](11_performance.md)** - AI load distribution details
- **[Chapter 16: Damage System](16_damage.md)** - How damage is calculated and applied
- **[Chapter 26: Visual Effects](26_effects.md)** - Explosion and death effects

---

## 8.17 Summary

Marathon's entity systems create a rich, dynamic game world through state machines and deterministic simulation:

**Monster System:**
- 47 monster types with faction relationships
- AI state machine for behavior
- Mode system for target tracking
- Load-distributed pathfinding and targeting

**Weapon System:**
- 10 weapons with complex state machines
- Dual-wield support
- Shell casing visual effects

**Projectile System:**
- 39 projectile types
- 24 damage types with immunities/weaknesses

**Support Systems:**
- Effects for visual feedback
- Motion sensor for tactical awareness
- Terminals for story delivery
- Recording/replay for demos
- Save/load for game persistence

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAXIMUM_MONSTERS_PER_MAP` | 512 | Monster limit |
| `MAXIMUM_PROJECTILES_PER_MAP` | 128 | Projectile limit |
| `MAXIMUM_EFFECTS_PER_MAP` | 64 | Effect limit |
| `MAXIMUM_PATHS` | 20 | Concurrent pathfinding |
| `MAXIMUM_POINTS_PER_PATH` | 63 | Waypoints per path |

### Key Source Files

| File | Purpose |
|------|---------|
| `monsters.c` | Monster AI and behavior |
| `monster_definitions.h` | Monster type data |
| `weapons.c` | Weapon state machine |
| `weapon_definitions.h` | Weapon type data |
| `projectiles.c` | Projectile physics |
| `effects.c` | Visual effects |
| `pathfinding.c` | Path computation |
| `flood_map.c` | Flood fill search |
| `motion_sensor.c` | Radar display |

---

*Next: [Chapter 9: Networking Architecture](09_networking.md) - Deterministic peer-to-peer multiplayer*
