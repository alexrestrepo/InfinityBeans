# Chapter 15: Control Panels

## Switches, Terminals, and Interactive Surfaces

> **For Porting:** Control panel logic in `devices.c` is portable. The terminal interface in `computer_interface.c` uses QuickDraw for text rendering—replace with your font/text system.

> **Source Files:** `devices.c` (812 lines), `computer_interface.c`, `map.h:450-461`

---

## 15.1 What Problem Are We Solving?

Marathon needs interactive wall surfaces that allow players to:

- **Recharge shields and oxygen** at stations
- **Activate doors and platforms** via switches
- **Save progress** at pattern buffers
- **Read story content** at computer terminals

The control panel system provides a unified interface for all wall-based interactivity.

---

## 15.2 Panel Classes

From `map.h:450-461`:
```c
enum {  // control_panel_class
    _panel_is_oxygen_refuel,       // 0: Restores oxygen
    _panel_is_shield_refuel,       // 1: Restores shields (1x)
    _panel_is_double_shield_refuel, // 2: Restores shields (2x speed)
    _panel_is_triple_shield_refuel, // 3: Restores shields (3x speed)
    _panel_is_light_switch,        // 4: Toggles light source
    _panel_is_platform_switch,     // 5: Activates platform/door
    _panel_is_tag_switch,          // 6: Triggers tagged objects
    _panel_is_pattern_buffer,      // 7: Save game point
    _panel_is_computer_terminal    // 8: Information terminal
};
```

---

## 15.3 Panel Definition Structure

Each panel type is defined by a static structure (from `devices.c:34-54`):

```c
struct control_panel_definition {
    short panel_class;              // Type from enum above
    word flags;                     // Behavior flags

    short collection;               // Texture collection
    short active_shape;             // Texture when active/on
    short inactive_shape;           // Texture when inactive/off

    short sounds[NUMBER_OF_CONTROL_PANEL_SOUNDS];  // 3 sounds
    fixed sound_frequency;          // Pitch modifier

    short item;                     // Required item (NONE = no requirement)
};
```

### Panel Definition Table

Marathon defines 48 control panel types across different environments (from `devices.c:66-148`):

| Index | Class | Collection | Environment |
|-------|-------|------------|-------------|
| 0-5 | Oxygen, Shield×3, Light, Platform | Water | Pfhor ship |
| 6-11 | Same | Lava | Lava environment |
| 12-17 | Same | Sewage | Sewage environment |
| 18-23 | Same | Jjaro | Jjaro technology |
| 24-29 | Same | Pfhor | Pfhor standard |
| 30-35 | Tag switches | Various | All environments |
| 36-41 | Pattern buffers | Various | All environments |
| 42-47 | Terminals | Various | All environments |

### Sound Indices

```c
enum {
    _activating_sound,    // 0: Sound when activated
    _deactivating_sound,  // 1: Sound when deactivated
    _unusable_sound,      // 2: Sound when can't use (no ammo, wrong item)
    NUMBER_OF_CONTROL_PANEL_SOUNDS
};
```

---

## 15.4 Activation System

### Activation Constants

```c
#define MAXIMUM_ACTIVATION_RANGE (3*WORLD_ONE)        // ~3072 units
#define MAXIMUM_CONTROL_ACTIVATION_RANGE (WORLD_ONE+WORLD_ONE_HALF)  // ~1536 units
#define MINIMUM_RESAVE_TICKS (2*TICKS_PER_SECOND)     // 60 ticks between saves
```

### Activation Flow

```
Player presses Action key:
    │
    ├─► find_action_key_target() [devices.c:177-280]
    │     │
    │     ├─► 1. Build list of nearby lines
    │     │       for each line in player's polygon and adjacent polygons:
    │     │         if distance < MAXIMUM_ACTIVATION_RANGE:
    │     │           add to candidate list
    │     │
    │     ├─► 2. Check each candidate line
    │     │       if line has control_panel_side_index != NONE:
    │     │         if distance < MAXIMUM_CONTROL_ACTIVATION_RANGE:
    │     │           if player is facing panel (dot product > 0):
    │     │             FOUND TARGET
    │     │
    │     └─► 3. Return target info
    │           (side_index, control_panel_index, target_type)
    │
    └─► change_panel_state() [devices.c:649-735]
          │
          ├─► Get control_panel_definition for this panel type
          │
          ├─► Switch on panel_class:
          │
          ├─► _panel_is_oxygen_refuel:
          │     └─► refuel_oxygen(player_index, panel_definition)
          │
          ├─► _panel_is_shield_refuel (1x, 2x, 3x):
          │     └─► refuel_shield(player_index, panel_definition, multiplier)
          │
          ├─► _panel_is_light_switch:
          │     ├─► Toggle light source state
          │     └─► Update panel texture (active ↔ inactive)
          │
          ├─► _panel_is_platform_switch:
          │     ├─► Toggle platform state (activate/deactivate)
          │     └─► Update panel texture
          │
          ├─► _panel_is_tag_switch:
          │     ├─► set_tagged_light_statuses(tag, state)
          │     ├─► try_and_change_tagged_platform_states(tag, state)
          │     └─► Update panel texture
          │
          ├─► _panel_is_pattern_buffer:
          │     ├─► Check cooldown (MINIMUM_RESAVE_TICKS)
          │     └─► save_game() if allowed
          │
          └─► _panel_is_computer_terminal:
                └─► enter_computer_interface(terminal_id)
```

### Panel State Machine

Panels have two states tracked by the side's `flags` field:

```c
// From map.h - side flags
#define _control_panel_status  0x0001   // 0 = inactive/off, 1 = active/on
```

**State Transitions:**

```
                    ┌───────────────────────────┐
                    │                           │
                    ▼                           │
            ┌──────────────┐                    │
            │   INACTIVE   │                    │
            │  (texture:   │                    │
            │   inactive)  │                    │
            └──────┬───────┘                    │
                   │                            │
         Player activates                       │
                   │                            │
                   ▼                            │
            ┌──────────────┐                    │
            │    ACTIVE    │                    │
            │  (texture:   │                    │
            │   active)    │                    │
            └──────┬───────┘                    │
                   │                            │
    ┌──────────────┼──────────────┐             │
    │              │              │             │
    ▼              ▼              ▼             │
 TOGGLE       MOMENTARY      ONE-SHOT           │
 switches     switches       switches           │
    │              │              │             │
    │         Held down:     Stays on           │
    │         stays active   permanently        │
    │              │              │             │
 Player        Released:          │             │
 reactivates   returns to         │             │
    │          inactive           │             │
    │              │              │             │
    └──────────────┴──────────────┴─────────────┘
```

---

## 15.5 Recharge Stations

### Recharge Implementation

From `devices.c:325-430`, recharging is a continuous process while the action key is held:

```c
void refuel_oxygen(short player_index, struct control_panel_definition *definition)
{
    struct player_data *player = get_player_data(player_index);

    // Calculate maximum oxygen based on difficulty
    short maximum_oxygen = MAXIMUM_SUIT_OXYGEN;

    if (player->suit_oxygen < maximum_oxygen) {
        // Add oxygen incrementally
        short oxygen_to_add = OXYGEN_RECHARGE_PER_TICK;

        player->suit_oxygen = MIN(player->suit_oxygen + oxygen_to_add, maximum_oxygen);

        // Play sound at start
        if (first_tick_of_recharge) {
            play_object_sound(player->object_index, definition->sounds[_activating_sound]);
        }
    } else {
        // Already full - play "unusable" sound
        play_object_sound(player->object_index, definition->sounds[_unusable_sound]);
    }
}
```

### Recharge Rates

From `devices.c:255-257`:

| Panel Type | Rate Per Tick | Notes |
|------------|---------------|-------|
| Oxygen Refuel | OXYGEN_RECHARGE_PER_TICK | ~Full in 1 second |
| Shield (1x) | FIXED_ONE | Normal speed |
| Shield (2x) | FIXED_ONE + FIXED_ONE/8 | 12.5% faster |
| Shield (3x) | FIXED_ONE + FIXED_ONE/4 | 25% faster |

### Shield Multiplier Calculation

```c
fixed get_recharge_rate(short panel_class)
{
    switch (panel_class) {
        case _panel_is_shield_refuel:
            return FIXED_ONE;                           // 65536

        case _panel_is_double_shield_refuel:
            return FIXED_ONE + (FIXED_ONE >> 3);        // 65536 + 8192 = 73728

        case _panel_is_triple_shield_refuel:
            return FIXED_ONE + (FIXED_ONE >> 2);        // 65536 + 16384 = 81920
    }
}
```

### Visualization

```
Oxygen Station:             Shield Station:
┌──────────────┐           ┌──────────────┐
│ ░░░░░░░░░░░░ │           │ ▓▓▓▓▓▓▓▓▓▓▓▓ │
│ ░░ O2   ░░░░ │           │ ▓▓ shld▓▓▓▓▓ │
│ ░░ REFUEL ░░ │           │ ▓▓ CHARGE ▓▓ │
│ ░░░░░░░░░░░░ │           │ ▓▓▓▓▓▓▓▓▓▓▓▓ │
└──────────────┘           └──────────────┘

Player holds Action:
  O2: ████████░░ → ██████████
  Shields: ███░░░░░░░ → █████████░
```

---

## 15.6 Tag System Interaction

Tag switches can trigger multiple objects simultaneously using Marathon's tag system.

### How Tags Work

```
Tag 1 assigned to:                    Tag switch activates Tag 1:
┌─────────────────────────────┐      ┌─────────────────────────────┐
│                             │      │                             │
│  [Light A]                  │      │  [Light A] ← ON             │
│  tag=1                      │      │                             │
│                             │      │  [Light B] ← ON             │
│  [Light B]                  │      │                             │
│  tag=1                      │  →   │  [Platform] ← ACTIVATED     │
│                             │      │                             │
│  [Platform]                 │      │                             │
│  tag=1                      │      │                             │
│                             │      │                             │
└─────────────────────────────┘      └─────────────────────────────┘
```

### Tag Switch Implementation

```c
// From devices.c:680-710
case _panel_is_tag_switch:
{
    short tag = side->control_panel_tag;
    boolean new_state = !GET_CONTROL_PANEL_STATUS(side);

    // Toggle all lights with this tag
    set_tagged_light_statuses(tag, new_state);

    // Toggle all platforms with this tag
    try_and_change_tagged_platform_states(tag, new_state);

    // Update panel visual state
    SET_CONTROL_PANEL_STATUS(side, new_state);

    // Update panel texture
    side->primary_texture.texture =
        new_state ? definition->active_shape : definition->inactive_shape;

    // Play appropriate sound
    play_control_panel_sound(side,
        new_state ? _activating_sound : _deactivating_sound);
}
```

### Tag Propagation Functions

**set_tagged_light_statuses()** (from `lights.c`):
```c
void set_tagged_light_statuses(short tag, boolean state)
{
    for (i = 0; i < dynamic_world->light_count; i++) {
        struct light_data *light = get_light_data(i);

        if (light->tag == tag) {
            set_light_status(i, state);
        }
    }
}
```

**try_and_change_tagged_platform_states()** (from `platforms.c`):
```c
boolean try_and_change_tagged_platform_states(short tag, boolean state)
{
    boolean changed = FALSE;

    for (i = 0; i < dynamic_world->platform_count; i++) {
        struct platform_data *platform = get_platform_data(i);

        if (platform->tag == tag) {
            if (state) {
                activate_platform(i);
            } else {
                deactivate_platform(i);
            }
            changed = TRUE;
        }
    }

    return changed;
}
```

---

## 15.7 Switch Types and Flags

### Panel Behavior Flags

```c
// From map.h
#define _control_panel_status             0x0001  // On/off state
#define _switch_control_panel             0x0002  // Toggle on each activation
#define _repair_switch                    0x0004  // Needs repair item
#define _can_only_be_hit_by_projectiles   0x0008  // Shootable switch
```

### Switch Behavior Matrix

| Flag Combination | Behavior |
|------------------|----------|
| None | One-shot activation (stays on) |
| `_switch_control_panel` | Toggle switch (on→off→on) |
| `_repair_switch` | Requires fusion repair item |
| `_can_only_be_hit_by_projectiles` | Must be shot, not pressed |

### Shootable Switches

Some switches must be shot with projectiles rather than pressed:

```c
// From devices.c:585-625 - damage_control_panel()
void damage_control_panel(short side_index)
{
    struct side_data *side = get_side_data(side_index);

    if (side->flags & _can_only_be_hit_by_projectiles) {
        // Treat as if player activated it
        change_panel_state(
            NONE,           // No specific player
            side_index,
            TRUE            // Force state change
        );
    }
}
```

---

## 15.8 Pattern Buffer (Save Points)

Pattern buffers implement in-game saves with cooldown protection.

### Save Cooldown

```c
#define MINIMUM_RESAVE_TICKS (2*TICKS_PER_SECOND)  // 60 ticks = 2 seconds

// From devices.c:720-740
case _panel_is_pattern_buffer:
{
    // Check if enough time has passed since last save
    if (game_time - last_save_time >= MINIMUM_RESAVE_TICKS) {
        // Save allowed
        save_game();
        last_save_time = game_time;
        play_control_panel_sound(side, _activating_sound);
    } else {
        // Too soon - play rejection sound
        play_control_panel_sound(side, _unusable_sound);
    }
}
```

### Save State Contents

Pattern buffers save complete game state (see `game_wad.c`):
- Player position, health, inventory
- Monster states and positions
- Platform/door states
- Light states
- Terminal visit flags
- Automap exploration

---

## 15.9 Computer Terminals

Computer terminals provide story content through an interactive text/image interface.

### Terminal Activation

```c
// From devices.c:742-760
case _panel_is_computer_terminal:
{
    short terminal_id = side->control_panel_permutation;

    // Enter terminal mode (pauses game)
    enter_computer_interface(terminal_id);
}
```

The terminal interface is covered in detail in **[Chapter 28: Terminals](28_terminals.md)**.

---

## 15.10 Summary

Control panels provide essential interactivity through a unified system:

- **9 panel classes** covering all interaction needs
- **Range-based activation** with facing requirement
- **Visual feedback** via texture changes
- **Continuous recharging** for oxygen/shields
- **Tag triggers** for complex level scripting
- **State persistence** across saves

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAXIMUM_ACTIVATION_RANGE` | 3×WORLD_ONE | Maximum interaction distance |
| `MAXIMUM_CONTROL_ACTIVATION_RANGE` | 1.5×WORLD_ONE | Precise panel distance |
| `MINIMUM_RESAVE_TICKS` | 60 | Pattern buffer cooldown |
| `NUMBER_OF_CONTROL_PANEL_SOUNDS` | 3 | Sound variants per panel |

### Key Source Files

| File | Purpose |
|------|---------|
| `devices.c` | Panel activation logic (812 lines) |
| `computer_interface.c` | Terminal display |
| `platforms.c` | Platform/door control |
| `lights.c` | Light state management |
| `map.h` | Panel class definitions |

### Source Reference Summary

| Function | Location | Purpose |
|----------|----------|---------|
| `find_action_key_target()` | devices.c:177 | Find targetable panel |
| `change_panel_state()` | devices.c:649 | Handle panel activation |
| `refuel_oxygen()` | devices.c:325 | Oxygen recharge logic |
| `damage_control_panel()` | devices.c:585 | Shootable switch handling |
| `set_tagged_light_statuses()` | lights.c | Tag propagation |

---

*Next: [Chapter 16: Damage System](16_damage.md) - Hit detection and damage types*
