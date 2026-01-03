# Chapter 23: View Bobbing & Camera System

## Player View, Weapon Sway, and Camera Effects

> **For Porting:** Camera calculations in `player.c` and `render.c` are portable. The view structure setup uses fixed-point math throughout.

---

## 23.1 What Problem Are We Solving?

The camera system must:

- **Track player position** - Where is the view origin?
- **Apply view bobbing** - Walking motion feels natural
- **Handle weapon sway** - First-person weapons move realistically
- **Support look up/down** - Vertical aiming
- **Manage special modes** - Extravision (wide FOV), terminal view

---

## 23.2 View Structure

```c
struct world_view {
    // Position
    world_point3d origin;           // Camera position in world
    short origin_polygon_index;     // Which polygon camera is in

    // Orientation
    angle yaw;                      // Horizontal facing (0-511)
    angle pitch;                    // Vertical look angle
    fixed virtual_yaw;              // Smoothed horizontal
    fixed virtual_pitch;            // Smoothed vertical

    // View bobbing
    fixed step_phase;               // Walk cycle phase (0 to 2π)
    fixed step_amplitude;           // Bob intensity
    world_distance step_delta;      // Vertical offset from bob

    // Rendering parameters
    short screen_width;             // Viewport width
    short screen_height;            // Viewport height
    angle half_cone;                // Half field-of-view
    short half_screen_width;        // screen_width / 2
    short half_screen_height;       // screen_height / 2

    // State
    short tick_count;               // Current game tick
    short shading_mode;             // Normal or infravision

    // Flags
    word flags;                     // _view_is_extravision, etc.
};
```

---

## 23.3 View Bobbing

### Bob Calculation

```c
#define STEP_AMPLITUDE (WORLD_ONE/24)        // Max vertical bob
#define STEP_PERIOD (FIXED_ONE*7/10)         // Bob cycle speed
#define BOB_PHASE_PERIOD (FIXED_ONE*2)       // Full cycle = 2π

void update_step_phase(struct player_data *player) {
    if (player_is_moving(player)) {
        // Advance phase based on movement
        player->step_phase += STEP_PERIOD;

        if (player->step_phase >= BOB_PHASE_PERIOD) {
            player->step_phase -= BOB_PHASE_PERIOD;
        }

        // Calculate vertical offset using sine
        // step_delta = amplitude * sin(phase)
        player->step_delta = FIXED_INTEGERAL_PART(
            STEP_AMPLITUDE * sine_table[
                (player->step_phase * NUMBER_OF_ANGLES / BOB_PHASE_PERIOD) & ANGULAR_MASK
            ]
        ) >> TRIG_SHIFT;
    } else {
        // Decay bob when stationary
        player->step_delta = (player->step_delta * 7) / 8;
        player->step_phase = 0;
    }
}
```

### Visualization

```
Walking View Bob:

Camera Height
    ↑
    │     ╱╲      ╱╲      ╱╲
    │    ╱  ╲    ╱  ╲    ╱  ╲
────│───╱────╲──╱────╲──╱────╲────► Time
    │  ╱      ╲╱      ╲╱      ╲
    │ ╱
    ↓

    └───────────────────────────┘
           One step cycle

Amplitude: ±WORLD_ONE/24 (~43 units)
Period: ~21 ticks per cycle
```

---

## 23.4 Field of View

### Normal vs Extravision

```c
#define NORMAL_FIELD_OF_VIEW 80           // Degrees
#define EXTRAVISION_FIELD_OF_VIEW 130     // With powerup

void calculate_view_cone(struct world_view *view) {
    short fov_degrees = view->flags & _view_is_extravision
        ? EXTRAVISION_FIELD_OF_VIEW
        : NORMAL_FIELD_OF_VIEW;

    // Convert to Marathon angle units (512 = 360°)
    view->half_cone = (fov_degrees * NUMBER_OF_ANGLES) / (2 * 360);
}
```

### FOV Comparison

```
Normal FOV (80°):                          Extravision FOV (130°):

        ╱│╲                                     ╱───│───╲
       ╱ │ ╲                                  ╱     │     ╲
      ╱  │  ╲                               ╱      │      ╲
     ╱   │   ╲                            ╱       │       ╲
    ╱    │    ╲                         ╱        │        ╲
   ╱     │     ╲                      ╱         │         ╲
  ╱      │      ╲                   ╱          │          ╲
 ╱       │       ╲                ╱           │           ╲
╱────────┼────────╲             ╱────────────┼────────────╲
    40°  │  40°                      65°     │     65°
         ▲                                   ▲
       Player                              Player

         Narrower view                       Much wider peripheral vision
      Objects appear larger               Objects appear smaller/compressed
```

---

## 23.5 Look Up/Down (Pitch)

### Pitch Limits

```c
#define MAXIMUM_PLAYER_ELEVATION (QUARTER_CIRCLE/3)  // ~30° up
#define MINIMUM_PLAYER_ELEVATION (-QUARTER_CIRCLE/3) // ~30° down

void update_player_elevation(struct player_data *player, long action_flags) {
    if (action_flags & _looking_up) {
        player->elevation += ANGULAR_VELOCITY;
        if (player->elevation > MAXIMUM_PLAYER_ELEVATION) {
            player->elevation = MAXIMUM_PLAYER_ELEVATION;
        }
    }

    if (action_flags & _looking_down) {
        player->elevation -= ANGULAR_VELOCITY;
        if (player->elevation < MINIMUM_PLAYER_ELEVATION) {
            player->elevation = MINIMUM_PLAYER_ELEVATION;
        }
    }

    // Auto-center when not looking
    if (!(action_flags & (_looking_up | _looking_down))) {
        player->elevation = (player->elevation * 7) / 8;  // Decay to center
    }
}
```

### Pitch Visualization

```
Looking Up (positive pitch):
         ╲___________________________╱   ← Far edge of ceiling (small)
          ╲                         ╱
           ╲      CEILING          ╱
            ╲                     ╱
             ╲                   ╱
              ╲                 ╱
               ╲_______________╱         ← Near edge (large, horizon)


Normal View (zero pitch):
             ╱───────────────────╲       ← Far ceiling (small)
            ╱                     ╲
           ╱       WALL            ╲
          ╱─────────────────────────╲    ← Horizon
          ╲                         ╱
           ╲       FLOOR           ╱
            ╲_____________________╱      ← Far floor (small)


Looking Down (negative pitch):
               ╱───────────────╲         ← Near edge (large, horizon)
              ╱                 ╲
             ╱                   ╲
            ╱       FLOOR        ╲
           ╱                       ╲
          ╱                         ╲
         ╱___________________________╲   ← Far edge of floor (small)
```

**Perspective Rule:** The part of the surface closer to you appears **larger** on screen. When looking up, the ceiling directly overhead is close (large); the ceiling near the horizon is far (small). When looking down, the floor directly below is close (large); the floor near the horizon is far (small).

---

## 23.6 Weapon Sway

First-person weapons respond to player movement:

### Sway Components

```c
struct weapon_display_information {
    short horizontal_offset;    // Side-to-side sway
    short vertical_offset;      // Up-down movement
    shape_descriptor shape;     // Weapon graphic
    short frame;               // Animation frame
};

void calculate_weapon_sway(
    struct player_data *player,
    struct weapon_display_information *info)
{
    // Horizontal sway from turning
    info->horizontal_offset = player->angular_velocity / SWAY_DIVISOR;

    // Vertical from walking bob
    info->vertical_offset = player->step_delta / 2;

    // Add recoil if firing
    if (player->weapon_state == _weapon_firing) {
        info->vertical_offset += RECOIL_OFFSET;
    }
}
```

### Sway Visualization

```
Weapon Position During Movement:

Stationary:          Turning Left:        Walking:
    ┌───────┐           ┌───────┐           ┌───────┐
    │       │           │       │           │       │
    │  ▄▄   │        ←  │ ▄▄    │           │  ▄▄   │  ↕
    │ ████  │           │████   │           │ ████  │
    │ ████  │           │████   │           │ ████  │
    └───────┘           └───────┘           └───────┘
                        (offset left)       (bobbing)

Firing:
    ┌───────┐
    │  ▄▄   │  ↑ recoil
    │ ████  │
    │ ████  │
    │       │  (kick up then return)
    └───────┘
```

---

## 23.7 View Setup for Rendering

```c
void setup_world_view(
    struct world_view *view,
    struct player_data *player,
    short tick_count)
{
    // Position with bob
    view->origin.x = player->location.x;
    view->origin.y = player->location.y;
    view->origin.z = player->location.z + player->step_delta;

    // Orientation
    view->yaw = player->facing;
    view->pitch = player->elevation;

    // State
    view->origin_polygon_index = player->polygon_index;
    view->tick_count = tick_count;

    // Special modes
    if (player->extravision_duration > 0) {
        view->flags |= _view_is_extravision;
    }

    if (player->infravision_duration > 0) {
        view->shading_mode = _shading_infravision;
    } else {
        view->shading_mode = _shading_normal;
    }

    // Calculate derived values
    calculate_view_cone(view);
}
```

---

## 23.8 Infravision Mode

```c
enum /* shading modes */ {
    _shading_normal,      // Standard lighting
    _shading_infravision  // See in darkness
};
```

### Visual Effect

```
Normal Mode:                    Infravision Mode:
┌─────────────────────┐        ┌─────────────────────┐
│░░░░░░░░░░░░░░░░░░░░░│        │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
│░░░░░░███░░░░░░░░░░░░│        │▓▓▓▓▓▓███▓▓▓▓▓▓▓▓▓▓▓▓│
│░░░░░░███░░░░░░░░░░░░│        │▓▓▓▓▓▓███▓▓▓▓▓▓▓▓▓▓▓▓│
│░░░░░░░░░░░░░░░░░░░░░│        │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│
└─────────────────────┘        └─────────────────────┘
  (dark areas hidden)            (all areas visible)

Monsters and objects rendered with enhanced brightness
Lasts approximately 60 seconds (powerup duration)
```

---

## 23.9 Camera Shake

For explosion effects:

```c
void apply_camera_shake(struct world_view *view, short intensity) {
    // Add random offset based on intensity
    view->origin.x += (random() % intensity) - intensity/2;
    view->origin.y += (random() % intensity) - intensity/2;
    view->origin.z += (random() % intensity) - intensity/2;

    // Small yaw perturbation
    view->yaw += (random() % (intensity/4)) - intensity/8;
}
```

---

## 23.10 Summary

Marathon's camera system provides:

- **View bobbing** with sinusoidal walk cycle
- **Pitch limits** for looking up/down
- **Weapon sway** responding to movement
- **FOV modes** (normal and extravision)
- **Visual modes** (normal and infravision)

### Key Source Files

| File | Purpose |
|------|---------|
| `player.c` | View bob, pitch updates |
| `render.c` | View structure setup |
| `weapons.c` | Weapon sway calculation |
| `physics.c` | Player movement affecting bob |

---

*Next: [Chapter 24: cseries.lib Utility Library](24_cseries.md) - Foundation types and utilities*
