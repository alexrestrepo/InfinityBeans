# Chapter 21: HUD Rendering System

## Health, Ammo, Motion Sensor, and Interface Panels

> **Source files**: `game_window.c`, `game_window.h`, `motion_sensor.c`, `motion_sensor.h`, `screen_drawing.h`
> **Related chapters**: [Chapter 14: Items](14_items.md), [Chapter 17: Multiplayer](17_multiplayer.md)

> **For Porting:** The HUD logic in `game_window.c` is mostly portable. Mac-specific drawing in `game_window_macintosh.c` needs replacement with your graphics API. The motion sensor code in `motion_sensor.c` is fully portable.

---

## 21.1 What Problem Are We Solving?

Players need visual feedback about their current state:

- **Health/Shields** - How much damage can I take?
- **Oxygen** - How long can I stay underwater?
- **Ammunition** - How many shots do I have left?
- **Motion Sensor** - Where are enemies and allies?
- **Inventory** - What items am I carrying?

**The constraints:**
- Minimize redraw to maintain framerate
- Display complex information compactly
- Support different weapon ammo types (bullets vs energy)

---

## 21.2 Interface Rectangle IDs (`screen_drawing.h:8-31`)

```c
enum {
    /* game window rectangles */
    _player_name_rect= 0,
    _oxygen_rect,
    _shield_rect,
    _motion_sensor_rect,
    _microphone_rect,
    _inventory_rect,
    _weapon_display_rect,

    /* interface rectangles */
    _new_game_button_rect,
    _load_game_button_rect,
    _gather_button_rect,
    _join_button_rect,
    _prefs_button_rect,
    _replay_last_button_rect,
    _save_last_button_rect,
    _replace_saved_button_rect,
    _credits_button_rect,
    _quit_button_rect,
    _center_button_rect,
    NUMBER_OF_INTERFACE_RECTANGLES
};
```

---

## 21.3 HUD Layout

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          3D GAME VIEW                                   │
│                                                                         │
│                                                                         │
│                                                                         │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                           HUD PANEL                                     │
│  ┌──────────┐   ┌─────────────────────────────┐   ┌──────────┐          │
│  │  MOTION  │   │       WEAPON PANEL          │   │  AMMO    │          │
│  │  SENSOR  │   │   ┌─────────────────────┐   │   │  DISPLAY │          │
│  │    ●     │   │   │    Weapon Name      │   │   │  ████░░  │          │
│  │   ╱│╲    │   │   │    [Graphic]        │   │   │  52/52   │          │
│  │    │     │   │   └─────────────────────┘   │   │          │          │
│  └──────────┘   └─────────────────────────────┘   └──────────┘          │
│  ┌──────────┐                                     ┌──────────┐          │
│  │ SHIELDS  │                                     │  OXYGEN  │          │
│  │ ████████ │                                     │ ████████ │          │
│  └──────────┘                                     └──────────┘          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 21.4 Interface State Management (`game_window.c:174-180`)

The HUD uses dirty flags to minimize redraw:

```c
struct interface_state_data
{
    boolean ammo_is_dirty;
    boolean weapon_is_dirty;
    boolean shield_is_dirty;
    boolean oxygen_is_dirty;
};
```

### Mark Functions (`game_window.h:14-21`)

```c
void mark_ammo_display_as_dirty(void);
void mark_shield_display_as_dirty(void);
void mark_oxygen_display_as_dirty(void);
void mark_weapon_display_as_dirty(void);
void mark_player_inventory_screen_as_dirty(short player_index, short screen);
void mark_player_inventory_as_dirty(short player_index, short dirty_item);
void mark_interface_collections(boolean loading);
void mark_player_network_stats_as_dirty(short player_index);
```

### Inventory Flag Macros (`game_window.c:63-74`)

```c
#define INVENTORY_MASK_BITS 0x0007
#define INVENTORY_DIRTY_BIT 0x0010
#define INTERFACE_DIRTY_BIT 0x0020

#define GET_CURRENT_INVENTORY_SCREEN(p) ((p)->interface_flags & INVENTORY_MASK_BITS)

#define INVENTORY_IS_DIRTY(p) ((p)->interface_flags & INVENTORY_DIRTY_BIT)
#define SET_INVENTORY_DIRTY_STATE(p, v) ((v)?((p)->interface_flags|=(word)INVENTORY_DIRTY_BIT):((p)->interface_flags&=(word)~INVENTORY_DIRTY_BIT))

#define INTERFACE_IS_DIRTY(p) ((p)->interface_flags & INTERFACE_DIRTY_BIT)
#define SET_INTERFACE_DIRTY_STATE(p, v) ((v)?((p)->interface_flags |= INTERFACE_DIRTY_BIT):(p)->interface_flags &= ~INTERFACE_DIRTY_BIT)
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

## 21.5 HUD Constants (`game_window.c:46-57`)

```c
#define TEXT_INSET 2
#define NAME_OFFSET 23
#define TOP_OF_BAR_WIDTH 8

#define MOTION_SENSOR_SIDE_LENGTH 123
#define DELAY_TICKS_BETWEEN_OXYGEN_REDRAW (2*TICKS_PER_SECOND)
#define RECORDING_LIGHT_FLASHING_DELAY (TICKS_PER_SECOND)

#define MICROPHONE_STOP_CLICK_SOUND ((short) 1250)
#define MICROPHONE_START_CLICK_SOUND ((short) 1280)

#define TOP_OF_BAR_HEIGHT 4
```

---

## 21.6 Energy Bar Shape IDs (`game_window.c:78-131`)

```c
enum {
    _empty_energy_bar=0,
    _energy_bar,
    _energy_bar_right,
    _double_energy_bar,
    _double_energy_bar_right,
    _triple_energy_bar,
    _triple_energy_bar_right,
    _empty_oxygen_bar,
    _oxygen_bar,
    _oxygen_bar_right,
    _motion_sensor_mount,
    _motion_sensor_virgin_mount,
    _motion_sensor_alien,
    _motion_sensor_friend= _motion_sensor_alien+6,
    _motion_sensor_enemy= _motion_sensor_friend+6,
    _network_panel= _motion_sensor_enemy+6,

    _magnum_bullet,
    _magnum_casing,
    _assault_rifle_bullet,
    _assault_rifle_casing,
    _alien_weapon_panel,
    _flamethrower_panel,
    _magnum_panel,
    _left_magnum,
    _zeus_panel,
    _assault_panel,
    _missile_panel,
    _left_magnum_unusable,
    _assault_rifle_grenade,
    _assault_rifle_grenade_casing,
    _shotgun_bullet,
    _shotgun_casing,
    _single_shotgun,
    _double_shotgun,
    _missile,
    _missile_casing,

    _network_compass_shape_nw,
    _network_compass_shape_ne,
    _network_compass_shape_sw,
    _network_compass_shape_se,

    _skull,

    _smg,
    _smg_bullet,
    _smg_casing,

    _mike_button_unpressed,
    _mike_button_pressed
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

---

## 21.7 Motion Sensor Constants (`motion_sensor.c:36-49`)

```c
#define MAXIMUM_MOTION_SENSOR_ENTITIES 12

#define NUMBER_OF_PREVIOUS_LOCATIONS 6

#define MOTION_SENSOR_UPDATE_FREQUENCY 5
#define MOTION_SENSOR_RESCAN_FREQUENCY 15

#define MOTION_SENSOR_RANGE (8*WORLD_ONE)

#define OBJECT_IS_VISIBLE_TO_MOTION_SENSOR(o) TRUE

#define MOTION_SENSOR_SCALE 7

#define FLICKER_FREQUENCY 0xf
```

### Entity Data Structure (`motion_sensor.c:60-74`)

```c
struct entity_data
{
    word flags; /* [slot_used.1] [slot_being_removed.1] [unused.14] */

    short monster_index;
    shape_descriptor shape;

    short remove_delay; /* only valid if this entity is being removed [0,NUMBER_OF_PREVIOUS_LOCATIONS) */

    point2d previous_points[NUMBER_OF_PREVIOUS_LOCATIONS];
    boolean visible_flags[NUMBER_OF_PREVIOUS_LOCATIONS];

    world_point3d last_location;
    angle last_facing;
};
```

### Motion Sensor Functions (`motion_sensor.h:6-14`)

```c
void initialize_motion_sensor(shape_descriptor mount, shape_descriptor virgin_mounts,
    shape_descriptor alien, shape_descriptor friendly, shape_descriptor enemy,
    shape_descriptor network_compass, short side_length);
void reset_motion_sensor(short monster_index);
void motion_sensor_scan(short ticks_elapsed);
boolean motion_sensor_has_changed(void);
void adjust_motion_sensor_range(void);
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
  - Scaled to fit sensor radius (>>MOTION_SENSOR_SCALE)
```

### Trail History

Each blip stores previous positions for motion trails (`motion_sensor.c:69-70`):

```c
point2d previous_points[NUMBER_OF_PREVIOUS_LOCATIONS];
boolean visible_flags[NUMBER_OF_PREVIOUS_LOCATIONS];
```

---

## 21.8 Weapon Interface Structures (`game_window.c:146-172`)

### Ammo Data (`game_window.c:146-158`)

```c
struct weapon_interface_ammo_data
{
    short type;              /* _unused_interface_data, _uses_energy, or _uses_bullets */
    short screen_left;       /* Position on screen */
    short screen_top;
    short ammo_across;       /* Bullets per row (or max energy for beam weapons) */
    short ammo_down;         /* Rows of bullets (unused for energy) */
    short delta_x;           /* Spacing (or width if uses energy) */
    short delta_y;           /* Spacing (or height if uses energy) */
    shape_descriptor bullet;        /* Full bullet icon (or fill color index) */
    shape_descriptor empty_bullet;  /* Empty bullet icon (or empty color index) */
    boolean right_to_left;   /* Draw direction */
};
```

### Weapon Interface Data (`game_window.c:160-172`)

```c
struct weapon_interface_data
{
    short item_id;                      /* Which weapon (from items.h) */
    short weapon_panel_shape;           /* Background graphic */
    short weapon_name_start_y;          /* Text position */
    short weapon_name_end_y;
    short weapon_name_start_x;          /* NONE means center */
    short weapon_name_end_x;            /* NONE means center */
    short standard_weapon_panel_top;    /* Panel position */
    short standard_weapon_panel_left;
    boolean multi_weapon;               /* Dual-wield capable? */
    struct weapon_interface_ammo_data ammo_data[NUMBER_OF_WEAPON_INTERFACE_ITEMS];
};
```

### Ammo Display Types (`game_window.c:133-137`)

```c
enum {
    _unused_interface_data,
    _uses_energy,      /* Energy bar (fusion pistol) */
    _uses_bullets      /* Bullet icons (pistol, rifle) */
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

## 21.9 Interface Colors (`screen_drawing.h:34-55`)

```c
enum {
    _energy_weapon_full_color,
    _energy_weapon_empty_color,
    _black_color,
    _inventory_text_color,
    _inventory_header_background_color,
    _inventory_background_color,
    PLAYER_COLOR_BASE_INDEX,

    _white_color= 14,
    _invalid_weapon_color,
    _computer_border_background_text_color,
    _computer_border_text_color,
    _computer_interface_text_color,
    _computer_interface_color_purple,
    _computer_interface_color_red,
    _computer_interface_color_pink,
    _computer_interface_color_aqua,
    _computer_interface_color_yellow,
    _computer_interface_color_brown,
    _computer_interface_color_blue
};
```

---

## 21.10 Network Compass (`game_window.c:117-120`)

In multiplayer, the compass shows objective directions:

```c
_network_compass_shape_nw,
_network_compass_shape_ne,
_network_compass_shape_sw,
_network_compass_shape_se,
```

### Compass Rendering (`motion_sensor.c:243-257`)

```c
static void draw_network_compass(
    void)
{
    short new_state= get_network_compass_state(motion_sensor_player_index);
    short difference= (new_state^network_compass_state)|new_state;

    if (difference&_network_compass_nw) draw_or_erase_unclipped_shape(36, 36, compass_shapes, new_state&_network_compass_nw);
    if (difference&_network_compass_ne) draw_or_erase_unclipped_shape(61, 36, compass_shapes+1, new_state&_network_compass_ne);
    if (difference&_network_compass_se) draw_or_erase_unclipped_shape(61, 61, compass_shapes+3, new_state&_network_compass_se);
    if (difference&_network_compass_sw) draw_or_erase_unclipped_shape(36, 61, compass_shapes+2, new_state&_network_compass_sw);

    network_compass_state= new_state;
}
```

### Compass Display

```
Objective Direction Indicator:

    ┌──────────────┐
    │ ░░░░ │ ▓▓▓▓  │  NE quadrant lit = objective northeast
    │──────┼───────│
    │ ░░░░ │ ░░░░  │
    └──────────────┘

Used for:
  - Flag location (CTF)
  - Ball carrier (Kill Man With Ball)
  - Hill location (King of the Hill)
  - "It" player (Tag)
```

---

## 21.11 Font System (`screen_drawing.h:67-76`)

```c
enum { /* Fonts for the interface et al.. */
    _interface_font,
    _weapon_name_font,
    _player_name_font,
    _interface_item_count_font,
    _computer_interface_font,
    _computer_interface_title_font,
    _net_stats_font,
    NUMBER_OF_INTERFACE_FONTS
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

### Justification Flags (`screen_drawing.h:57-65`)

```c
enum { /* justification flags for _draw_screen_text */
    _no_flags,
    _center_horizontal= 0x01,
    _center_vertical= 0x02,
    _right_justified= 0x04,
    _top_justified= 0x08,
    _bottom_justified= 0x10,
    _wrap_text= 0x20
};
```

### Screen Rectangle Structure (`screen_drawing.h:79-83`)

```c
struct screen_rectangle {
    short top, left;
    short bottom, right;
};
typedef struct screen_rectangle screen_rectangle;
```

---

## 21.12 Motion Sensor Scan (`motion_sensor.c:179-221`)

```c
void motion_sensor_scan(
    short ticks_elapsed)
{
    struct object_data *owner_object= get_object_data(get_player_data(motion_sensor_player_index)->object_index);

    /* if we need to scan for new objects, flood around the owner monster looking for other,
        visible monsters within our range */
    if ((ticks_since_last_rescan-= ticks_elapsed)<0 || ticks_elapsed==NONE)
    {
        struct monster_data *monster;
        short monster_index;

        for (monster_index=0,monster=monsters;monster_index<MAXIMUM_MONSTERS_PER_MAP;++monster,++monster_index)
        {
            if (SLOT_IS_USED(monster)&&(MONSTER_IS_PLAYER(monster)||MONSTER_IS_ACTIVE(monster)))
            {
                struct object_data *object= get_object_data(monster->object_index);
                world_distance distance= guess_distance2d((world_point2d *) &object->location, (world_point2d *) &owner_object->location);

                if (distance<MOTION_SENSOR_RANGE && OBJECT_IS_VISIBLE_TO_MOTION_SENSOR(object))
                {
                    find_or_add_motion_sensor_entity(object->permutation);
                }
            }
        }

        ticks_since_last_rescan= MOTION_SENSOR_RESCAN_FREQUENCY;
    }

    /* if we need to update the motion sensor, draw all active entities */
    if ((ticks_since_last_update-= ticks_elapsed)<0 || ticks_elapsed==NONE)
    {
        erase_all_entity_blips();
        if (dynamic_world->player_count>1) draw_network_compass();
        draw_all_entity_blips();

        ticks_since_last_update= MOTION_SENSOR_UPDATE_FREQUENCY;
        motion_sensor_changed= TRUE;
    }
}
```

---

## 21.13 Entity Shape Selection (`motion_sensor.c:613-649`)

```c
static shape_descriptor get_motion_sensor_entity_shape(
    short monster_index)
{
    struct monster_data *monster= get_monster_data(monster_index);
    shape_descriptor shape;

    if (MONSTER_IS_PLAYER(monster))
    {
        struct player_data *player= get_player_data(monster_index_to_player_index(monster_index));
        struct player_data *owner= get_player_data(motion_sensor_player_index);

        shape= ((player->team==owner->team && !(GET_GAME_OPTIONS()&_force_unique_teams)) || GET_GAME_TYPE()==_game_of_cooperative_play) ?
            friendly_shapes : enemy_shapes;
    }
    else
    {
        switch (monster->type)
        {
            case _civilian_crew:
            case _civilian_science:
            case _civilian_security:
            case _civilian_assimilated:
            case _vacuum_civilian_crew:
            case _vacuum_civilian_science:
            case _vacuum_civilian_security:
            case _vacuum_civilian_assimilated:
                shape= friendly_shapes;
                break;

            default:
                shape= alien_shapes;
                break;
        }
    }

    return shape;
}
```

---

## 21.14 Summary

Marathon's HUD system provides:

- **Dirty-flag rendering** for efficiency (`game_window.c:174-180`)
- **Flexible energy bars** (1x, 2x, 3x shields) (`game_window.c:78-87`)
- **Motion sensor** with entity trails (`motion_sensor.c:36-74`)
- **Weapon-specific** ammo displays (`game_window.c:146-172`)
- **Network compass** for multiplayer objectives (`motion_sensor.c:243-257`)

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `MOTION_SENSOR_SIDE_LENGTH` | 123 | `game_window.c:50` |
| `MAXIMUM_MOTION_SENSOR_ENTITIES` | 12 | `motion_sensor.c:36` |
| `MOTION_SENSOR_RANGE` | 8×WORLD_ONE | `motion_sensor.c:43` |
| `NUMBER_OF_PREVIOUS_LOCATIONS` | 6 | `motion_sensor.c:38` |
| `DELAY_TICKS_BETWEEN_OXYGEN_REDRAW` | 60 | `game_window.c:51` |
| `MOTION_SENSOR_UPDATE_FREQUENCY` | 5 | `motion_sensor.c:40` |
| `MOTION_SENSOR_RESCAN_FREQUENCY` | 15 | `motion_sensor.c:41` |
| `NUMBER_OF_INTERFACE_FONTS` | 7 | `screen_drawing.h:75` |

### Key Source Files

| File | Purpose |
|------|---------|
| `game_window.c` | HUD logic, state, weapon interface definitions |
| `game_window.h` | Interface function prototypes (lines 8-24) |
| `game_window_macintosh.c` | Mac drawing code |
| `motion_sensor.c` | Radar implementation (650 lines) |
| `motion_sensor.h` | Sensor function prototypes (lines 6-14) |
| `screen_drawing.h` | Rectangles, colors, fonts (lines 8-83) |
| `screen_drawing.c` | Font rendering, text layout |

---

## 21.15 See Also

- [Chapter 14: Items](14_items.md) — Inventory items displayed in HUD
- [Chapter 17: Multiplayer](17_multiplayer.md) — Network compass state calculation
- [Chapter 22: Fades](22_fades.md) — Damage flashes and transitions

---

*Next: [Chapter 22: Screen Effects & Fades](22_fades.md) - Damage flashes and transitions*
