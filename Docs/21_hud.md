# Chapter 21: HUD Rendering System

## Health, Ammo, Motion Sensor, and Interface Panels

> **For Porting:** The HUD logic in `game_window.c` is mostly portable. Mac-specific drawing in `game_window_macintosh.c` needs replacement with your graphics API. The motion sensor code in `motion_sensor.c` is fully portable.

---

## 21.1 What Problem Are We Solving?

Players need visual feedback about their current state:

- **Health/Shields** - How much damage can I take?
- **Oxygen** - How long can I stay underwater?
- **Ammunition** - How many shots do I have left?
- **Motion Sensor** - Where are enemies and allies?
- **Inventory** - What items am I carrying?

---

## 21.2 HUD Layout

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          3D GAME VIEW                                    │
│                                                                          │
│                                                                          │
│                                                                          │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                           HUD PANEL                                      │
│  ┌──────────┐   ┌─────────────────────────────┐   ┌──────────┐         │
│  │  MOTION  │   │       WEAPON PANEL          │   │  AMMO    │         │
│  │  SENSOR  │   │   ┌─────────────────────┐   │   │  DISPLAY │         │
│  │    ●     │   │   │    Weapon Name      │   │   │  ████░░  │         │
│  │   ╱│╲    │   │   │    [Graphic]        │   │   │  52/52   │         │
│  │    │     │   │   └─────────────────────┘   │   │          │         │
│  └──────────┘   └─────────────────────────────┘   └──────────┘         │
│  ┌──────────┐                                     ┌──────────┐         │
│  │ SHIELDS  │                                     │  OXYGEN  │         │
│  │ ████████ │                                     │ ████████ │         │
│  └──────────┘                                     └──────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 21.3 Interface State Management

The HUD uses dirty flags to minimize redraw:

```c
struct interface_state_data {
    boolean ammo_is_dirty;
    boolean weapon_is_dirty;
    boolean shield_is_dirty;
    boolean oxygen_is_dirty;
};

// Mark functions called when state changes
void mark_ammo_display_as_dirty(void);
void mark_shield_display_as_dirty(void);
void mark_oxygen_display_as_dirty(void);
void mark_weapon_display_as_dirty(void);
```

### Update Flow

```
Game Event                  Dirty Flag Set           Redraw on Next Frame
─────────────              ──────────────           ────────────────────
Player takes damage    →   shield_is_dirty = true   →   Redraw shield bar
Player fires weapon    →   ammo_is_dirty = true     →   Redraw ammo count
Player switches weapon →   weapon_is_dirty = true   →   Redraw weapon panel
Player underwater      →   oxygen_is_dirty = true   →   Redraw oxygen bar
```

---

## 21.4 Energy Bars

### Shield Bar Rendering

```c
enum {
    _empty_energy_bar = 0,
    _energy_bar,
    _energy_bar_right,
    _double_energy_bar,       // 2x shields
    _double_energy_bar_right,
    _triple_energy_bar,       // 3x shields
    _triple_energy_bar_right,
    // ...
};
```

### Bar Visualization

```
Shield Levels:

Single (0-100%):
████████████████████░░░░░░░░░░  (75% full)

Double (100-200%):
████████████████████████████████████████░░░░░░░░░░  (175% full)
└────── First bar ──────┘└─── Second bar ───┘

Triple (200-300%):
████████████████████████████████████████████████████████████░░░░░░  (275%)
└────── First bar ──────┘└─── Second bar ───┘└── Third bar ──┘
```

### Oxygen Bar

```c
#define DELAY_TICKS_BETWEEN_OXYGEN_REDRAW (2*TICKS_PER_SECOND)  // 60 ticks

// Oxygen only redraws every 2 seconds to reduce flicker
// when rapidly entering/leaving water
```

---

## 21.5 Motion Sensor

The motion sensor is a radar display showing nearby entities.

### Constants

```c
#define MOTION_SENSOR_SIDE_LENGTH 123        // Pixel size
#define MAXIMUM_MOTION_SENSOR_ENTITIES 12    // Max tracked blips
#define MOTION_SENSOR_RANGE (8*WORLD_ONE)    // Detection radius
#define NUMBER_OF_PREVIOUS_LOCATIONS 6       // Trail length
```

### Entity Classification

```c
enum /* motion sensor blip types */ {
    _motion_sensor_alien,     // Hostile (red)
    _motion_sensor_friend,    // Friendly (green)
    _motion_sensor_enemy      // Player enemy in multiplayer (yellow)
};

// Shape offsets in interface collection
_motion_sensor_mount,        // Background
_motion_sensor_virgin_mount, // Clean background
_motion_sensor_alien,        // Red blips (6 frames)
_motion_sensor_friend = _motion_sensor_alien + 6,   // Green blips
_motion_sensor_enemy = _motion_sensor_friend + 6,   // Yellow blips
```

### Blip Rendering

```
Motion Sensor Display:

        ┌─────────────────────┐
        │         N           │
        │    ●                │  ● = Alien (red)
        │         ○           │  ○ = Friend (green)
        │    ▲                │  ▲ = Player (center)
        │              ◆      │  ◆ = Enemy player (yellow)
        │         S           │
        └─────────────────────┘

Blip positions calculated from:
  - Distance from player (radial)
  - Angle relative to player facing
  - Scaled to fit sensor radius
```

### Trail History

Each blip stores previous positions for motion trails:

```c
struct motion_sensor_blip {
    world_point2d position;
    world_point2d previous_positions[NUMBER_OF_PREVIOUS_LOCATIONS];
    short type;                    // alien/friend/enemy
    short object_index;            // Which entity this tracks
    boolean visible;
};
```

---

## 21.6 Weapon Panel

### Weapon Interface Data

```c
struct weapon_interface_data {
    short item_id;                      // Which weapon
    short weapon_panel_shape;           // Background graphic
    short weapon_name_start_y;          // Text position
    short weapon_name_end_y;
    short weapon_name_start_x;          // NONE = center
    short weapon_name_end_x;
    short standard_weapon_panel_top;    // Panel position
    short standard_weapon_panel_left;
    boolean multi_weapon;               // Dual-wield capable?
    struct weapon_interface_ammo_data ammo_data[2];  // Primary/secondary
};
```

### Ammo Display Types

```c
enum {
    _unused_interface_data,
    _uses_energy,      // Energy bar (fusion pistol)
    _uses_bullets      // Bullet icons (pistol, rifle)
};

struct weapon_interface_ammo_data {
    short type;              // Energy or bullets
    short screen_left;       // Position
    short screen_top;
    short ammo_across;       // Bullets per row (or max energy)
    short ammo_down;         // Rows of bullets
    short delta_x;           // Spacing (or bar width)
    short delta_y;           // Spacing (or bar height)
    shape_descriptor bullet;        // Full bullet icon
    shape_descriptor empty_bullet;  // Empty bullet icon
    boolean right_to_left;   // Draw direction
};
```

### Ammo Visualization

```
Bullet-Based (Magnum):          Energy-Based (Fusion):
┌─────────────────────┐         ┌─────────────────────┐
│ ● ● ● ● ● ● ● ○     │         │ ████████████░░░░░░░ │
│ 8 rounds loaded     │         │ 75% charge          │
└─────────────────────┘         └─────────────────────┘

● = loaded round
○ = empty chamber
█ = energy remaining
░ = energy depleted
```

---

## 21.7 Network Compass

In multiplayer, the compass shows objective directions:

```c
enum {
    _network_compass_shape_nw,
    _network_compass_shape_ne,
    _network_compass_shape_sw,
    _network_compass_shape_se
};
```

### Compass Display

```
Objective Direction Indicator:

    ┌───────────────┐
    │ ░░░░ │ ▓▓▓▓  │  NE quadrant lit = objective northeast
    │──────┼───────│
    │ ░░░░ │ ░░░░  │
    └───────────────┘

Used for:
  - Flag location (CTF)
  - Ball carrier (Kill Man With Ball)
  - Hill location (King of the Hill)
  - "It" player (Tag)
```

---

## 21.8 Inventory Screens

```c
#define GET_CURRENT_INVENTORY_SCREEN(p) ((p)->interface_flags & INVENTORY_MASK_BITS)

enum /* inventory screens */ {
    _weapon_display,      // Current weapon
    _ammunition_display,  // All ammo counts
    _inventory_display    // Items (keys, powerups)
};
```

### Screen Switching

```c
void scroll_inventory(short dy) {
    // Called when player scrolls inventory
    // Cycles through: weapon → ammo → items → weapon
}
```

---

## 21.9 Interface Update Cycle

```c
void update_interface(short time_elapsed) {
    // Called each tick from game loop

    if (interface_state.shield_is_dirty) {
        draw_shield_bar();
        interface_state.shield_is_dirty = false;
    }

    if (interface_state.oxygen_is_dirty) {
        // Rate-limit oxygen updates
        if (tick_count - last_oxygen_draw > DELAY_TICKS_BETWEEN_OXYGEN_REDRAW) {
            draw_oxygen_bar();
            interface_state.oxygen_is_dirty = false;
        }
    }

    if (interface_state.ammo_is_dirty) {
        draw_ammo_display();
        interface_state.ammo_is_dirty = false;
    }

    if (interface_state.weapon_is_dirty) {
        draw_weapon_panel();
        interface_state.weapon_is_dirty = false;
    }

    // Motion sensor updates every frame
    update_motion_sensor();
}
```

---

## 21.10 Font System

The font system provides text rendering for all 2D interface elements (HUD, terminals, menus). Marathon does NOT render 3D in-world text.

### Font Storage Structure

```c
struct interface_font_info {
    TextSpec fonts[NUMBER_OF_INTERFACE_FONTS];      // Mac font specs (ID, size, style)
    short heights[NUMBER_OF_INTERFACE_FONTS];       // Cached ascent + leading
    short line_spacing[NUMBER_OF_INTERFACE_FONTS];  // Cached ascent + descent + leading
};

static struct interface_font_info interface_fonts;  // Global font cache
```

### Interface Fonts

```c
enum { /* Fonts for the interface */
    _interface_font,              // 0 - General UI text
    _weapon_name_font,            // 1 - Weapon names in HUD
    _player_name_font,            // 2 - Player names (multiplayer)
    _interface_item_count_font,   // 3 - Item/ammo counts
    _computer_interface_font,     // 4 - Terminal body text
    _computer_interface_title_font, // 5 - Terminal titles
    _net_stats_font,              // 6 - Network game statistics
    NUMBER_OF_INTERFACE_FONTS     // 7
};
```

| ID | Font Constant | Used For |
|----|---------------|----------|
| 0 | `_interface_font` | General UI, menus |
| 1 | `_weapon_name_font` | Weapon display |
| 2 | `_player_name_font` | Multiplayer names |
| 3 | `_interface_item_count_font` | Ammo/item numbers |
| 4 | `_computer_interface_font` | Terminal text |
| 5 | `_computer_interface_title_font` | Terminal headers |
| 6 | `_net_stats_font` | Network stats |

### Justification Flags

```c
enum { /* justification flags for _draw_screen_text */
    _no_flags           = 0x00,
    _center_horizontal  = 0x01,  // Center text horizontally in rect
    _center_vertical    = 0x02,  // Center text vertically in rect
    _right_justified    = 0x04,  // Align right edge
    _top_justified      = 0x08,  // Align to top
    _bottom_justified   = 0x10,  // Align to bottom
    _wrap_text          = 0x20   // Auto-wrap at word boundaries
};
```

### Usage by System

| System | Fonts Used | Purpose |
|--------|------------|---------|
| HUD (`game_window.c`) | 1, 3 | Weapon name, ammo count |
| Terminals (`computer_interface.c`) | 4, 5 | Body text, titles |
| Overhead Map (`overhead_map.c`) | 0 | Level annotations |
| Network (`network_dialogs.c`) | 2, 6 | Player names, stats |

### Porting Considerations

The font system is **entirely Mac-specific**, using QuickDraw and Font Manager.

**Porting Options**:
1. **Bitmap font**: Extract Marathon fonts as sprite sheets
2. **stb_truetype**: Load TrueType fonts for modern rendering
3. **Simple option**: Use fixed-width bitmap font (8x16 pixels per character)

**Key Functions to Replace**:
- `_draw_screen_text()` → Custom text renderer
- `_text_width()` → Calculate string width from glyph widths
- `_get_font_line_height()` → Return fixed line height

---

## 21.11 Summary

Marathon's HUD system provides:

- **Dirty-flag rendering** for efficiency
- **Flexible energy bars** (1x, 2x, 3x shields)
- **Motion sensor** with entity trails
- **Weapon-specific** ammo displays
- **Network compass** for multiplayer objectives

### Key Source Files

| File | Purpose |
|------|---------|
| `game_window.c` | HUD logic and state |
| `game_window.h` | Interface declarations |
| `game_window_macintosh.c` | Mac drawing code |
| `motion_sensor.c` | Radar implementation |
| `motion_sensor.h` | Sensor structures |
| `screen_drawing.c` | Font rendering, text layout |
| `screen_drawing.h` | Font and color definitions |

---

*Next: [Chapter 22: Screen Effects & Fades](22_fades.md) - Damage flashes and transitions*
