# Chapter 15: Control Panels

## Switches, Terminals, and Interactive Surfaces

> **For Porting:** Control panel logic in `control_panels.c` is portable. The terminal interface in `computer_interface.c` uses QuickDraw for text rendering—replace with your font/text system.

---

## 15.1 What Problem Are We Solving?

Marathon needs interactive wall surfaces that allow players to:

- **Recharge shields and oxygen** at stations
- **Activate doors and platforms** via switches
- **Save progress** at pattern buffers
- **Read story content** at computer terminals

---

## 15.2 Panel Classes

```c
enum {  // control_panel_class
    _panel_is_oxygen_refuel,       // Restores oxygen
    _panel_is_shield_refuel,       // Restores shields (1x)
    _panel_is_double_shield_refuel, // Restores shields (2x speed)
    _panel_is_triple_shield_refuel, // Restores shields (3x speed)
    _panel_is_light_switch,        // Toggles light source
    _panel_is_platform_switch,     // Activates platform/door
    _panel_is_tag_switch,          // Triggers tagged objects
    _panel_is_pattern_buffer,      // Save game point
    _panel_is_computer_terminal    // Information terminal
};
```

---

## 15.3 Panel Definition Structure

```c
struct control_panel_definition {
    short panel_class;              // Type from above
    word flags;                     // Behavior flags

    short collection;               // Texture collection
    short active_shape;             // Texture when active/on
    short inactive_shape;           // Texture when inactive/off

    short sounds[3];                // Activating, Deactivating, Unusable
    fixed sound_frequency;          // Pitch modifier

    short item;                     // Required item (NONE = no requirement)
};
```

---

## 15.4 Activation System

### Activation Constants

```c
#define MAXIMUM_ACTIVATION_RANGE (3*WORLD_ONE)
#define MAXIMUM_CONTROL_ACTIVATION_RANGE (WORLD_ONE+WORLD_ONE_HALF)  // ~1536 units
#define MINIMUM_RESAVE_TICKS (2*TICKS_PER_SECOND)  // Pattern buffer cooldown
```

### Activation Flow

```
Player presses Action key:
    │
    ├─► find_action_key_target()
    │     ├─► Search lines within range
    │     ├─► Check line has control panel
    │     └─► Verify player facing panel
    │
    └─► change_panel_state()
          │
          ├─► _panel_is_oxygen_refuel:
          │     └─► Add oxygen each tick while holding action
          │
          ├─► _panel_is_shield_refuel:
          │     └─► Add shields each tick (rate varies)
          │
          ├─► _panel_is_light_switch:
          │     └─► Toggle light on/off
          │
          ├─► _panel_is_platform_switch:
          │     └─► Activate/deactivate platform
          │
          ├─► _panel_is_tag_switch:
          │     └─► Trigger all objects with matching tag
          │
          ├─► _panel_is_pattern_buffer:
          │     └─► Save game state
          │
          └─► _panel_is_computer_terminal:
                └─► Enter terminal interface mode
```

---

## 15.5 Recharge Stations

### Recharge Rates

| Panel Type | Rate | Notes |
|------------|------|-------|
| Oxygen Refuel | Full/second | Restores O2 |
| Shield (1x) | FIXED_ONE/tick | Normal speed |
| Shield (2x) | FIXED_ONE+FIXED_ONE/8 | 12.5% faster |
| Shield (3x) | FIXED_ONE+FIXED_ONE/4 | 25% faster |

### Visualization

```
Oxygen Station:             Shield Station:
┌──────────────┐           ┌──────────────┐
│ ░░░░░░░░░░░░ │           │ ▓▓▓▓▓▓▓▓▓▓▓▓ │
│ ░░ O2  ░░░░ │           │ ▓▓ ⚡ ▓▓▓▓▓ │
│ ░░ REFUEL░░ │           │ ▓▓ CHARGE ▓▓ │
│ ░░░░░░░░░░░░ │           │ ▓▓▓▓▓▓▓▓▓▓▓▓ │
└──────────────┘           └──────────────┘

Player holds Action:
  O2: ████████░░ → ██████████
  Shields: ███░░░░░░░ → █████████░
```

---

## 15.6 Switch States

Switches maintain visual state based on activation:

```c
void set_control_panel_texture(struct side_data *side) {
    // Update texture based on current state
    // active_shape if ON, inactive_shape if OFF
}
```

---

## 15.7 Summary

Control panels provide essential interactivity:

- **9 panel types** covering all interaction needs
- **Range-based activation** with facing requirement
- **Visual feedback** via texture changes
- **Tag triggers** for complex level scripting

### Key Source Files

| File | Purpose |
|------|---------|
| `control_panels.c` | Panel activation logic |
| `computer_interface.c` | Terminal display |
| `platforms.c` | Platform/door control |

---

*Next: [Chapter 16: Damage System](16_damage.md) - Hit detection and damage types*
