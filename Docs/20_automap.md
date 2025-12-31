# Chapter 20: Automap System

## Overhead Map and Exploration Tracking

> **For Porting:** The automap logic in `overhead_map.c` is portable. Only the drawing primitives need replacement—swap QuickDraw calls with your graphics library.

---

## 20.1 What Problem Are We Solving?

Marathon's levels are complex 3D environments. Players need:

- **Navigation aid** - Know where they've been and where to go
- **Exploration tracking** - Remember which areas are visited
- **Entity awareness** - See enemies and items on the map
- **Mission context** - Terminal checkpoints show objectives

---

## 20.2 Exploration Tracking

Marathon tracks exploration with compact bitmask arrays:

```c
extern byte *automap_lines;     // 1 bit per line
extern byte *automap_polygons;  // 1 bit per polygon

// Access macros - store 8 lines/polygons per byte
#define ADD_LINE_TO_AUTOMAP(i) \
    (automap_lines[(i)>>3] |= (byte) 1<<((i)&0x07))

#define LINE_IS_IN_AUTOMAP(i) \
    ((automap_lines[(i)>>3]&((byte)1<<((i)&0x07)))?(TRUE):(FALSE))

#define ADD_POLYGON_TO_AUTOMAP(i) \
    (automap_polygons[(i)>>3] |= (byte) 1<<((i)&0x07))

#define POLYGON_IS_IN_AUTOMAP(i) \
    ((automap_polygons[(i)>>3]&((byte)1<<((i)&0x07)))?(TRUE):(FALSE))
```

### Memory Layout

```
For a level with 1000 lines and 500 polygons:

automap_lines:    [byte 0][byte 1][byte 2]...[byte 124] = 125 bytes
                   ↓
                  bits 0-7 for lines 0-7
                  bits 8-15 for lines 8-15
                  etc.

automap_polygons: [byte 0][byte 1]...[byte 62] = 63 bytes
                   ↓
                  bits 0-7 for polygons 0-7
                  etc.

Total: ~188 bytes for exploration state (very compact!)
```

### Exploration Triggers

- When player enters a polygon: `ADD_POLYGON_TO_AUTOMAP(polygon_index)`
- When player can see a line: `ADD_LINE_TO_AUTOMAP(line_index)`
- Rendering marks endpoints with flag `_endpoint_on_automap = 0x2000`

---

## 20.3 Overhead Map Data Structure

```c
struct overhead_map_data {
    short mode;                    // Rendering mode
    short scale;                   // Zoom level [1-4]
    world_point2d origin;          // Center of view
    short origin_polygon_index;    // Which polygon origin is in
    short half_width, half_height; // Half dimensions
    short width, height;           // Full dimensions
    short top, left;               // Screen position
    boolean draw_everything;       // Debug mode - show all
};
```

---

## 20.4 Rendering Modes

```c
enum /* overhead map modes */ {
    _rendering_saved_game_preview,  // Thumbnail in save dialog
    _rendering_checkpoint_map,      // Terminal checkpoint display
    _rendering_game_map             // Live gameplay map
};
```

| Mode | Shows Explored Only | Shows Player | Shows Entities | Interactive |
|------|---------------------|--------------|----------------|-------------|
| `_rendering_game_map` | Yes | Yes | Yes | Yes |
| `_rendering_checkpoint_map` | Uses false automap | No | Checkpoint only | No |
| `_rendering_saved_game_preview` | Yes | Yes | No | No |

---

## 20.5 Scale System

```c
#define OVERHEAD_MAP_MINIMUM_SCALE 1   // Zoomed out (overview)
#define OVERHEAD_MAP_MAXIMUM_SCALE 4   // Zoomed in (detail)
#define DEFAULT_OVERHEAD_MAP_SCALE 3

#define WORLD_TO_SCREEN_SCALE_ONE 8

// Convert world coordinates to screen coordinates
#define WORLD_TO_SCREEN(x, x0, scale) \
    (((x)-(x0))>>(WORLD_TO_SCREEN_SCALE_ONE-(scale)))
```

### Scale Visualization

```
Scale 1 (most zoomed out):
    Shift right by 7 bits → 128:1 reduction
    Large area visible, small details

Scale 4 (most zoomed in):
    Shift right by 4 bits → 16:1 reduction
    Small area visible, precise navigation

World coordinate 8192 with origin 4096:
    Scale 1: (8192-4096) >> 7 = 32 pixels from center
    Scale 4: (8192-4096) >> 4 = 256 pixels from center
```

---

## 20.6 Color System

### Polygon Colors (area fills)

```c
enum /* polygon colors */ {
    _polygon_color,           // Normal floor - dark gray
    _polygon_platform_color,  // Elevator/door - medium gray
    _polygon_water_color,     // Water - blue tint
    _polygon_lava_color,      // Lava - red/orange tint
    _polygon_goo_color,       // Goo/sewage - green tint
    _polygon_sewage_color,    // Sewage - brown tint
    _polygon_hill_color       // KOTH hill - special highlight
};
```

### Line Colors (edges)

```c
enum /* line colors */ {
    _solid_line_color,         // Impassable walls - bright
    _elevation_line_color,     // Height change - medium
    _control_panel_line_color  // Interactive panels - highlighted
};
```

### Entity Colors (things)

```c
enum /* thing colors */ {
    _civilian_thing,    // Friendly BOBs - green
    _item_thing,        // Pickups - yellow
    _monster_thing,     // Enemies - red
    _projectile_thing,  // Bullets/rockets - white
    _checkpoint_thing   // Terminal checkpoint - special
};
```

---

## 20.7 Rendering Pipeline

```
overhead_map_begin():
    │
    ├─► Save graphics state
    ├─► Set clipping rectangle
    └─► Clear map area to background color

draw_overhead_map():
    │
    ├─► Step 1: Draw polygons (area fills)
    │     for (each polygon in level):
    │         if (POLYGON_IS_IN_AUTOMAP(i) || draw_everything):
    │             determine polygon color from type
    │             transform vertices to screen
    │             fill polygon shape
    │
    ├─► Step 2: Draw lines (edges)
    │     for (each line in level):
    │         if (LINE_IS_IN_AUTOMAP(i) || draw_everything):
    │             determine line color from flags
    │             transform endpoints to screen
    │             draw line segment
    │
    ├─► Step 3: Draw things (entities)
    │     for (each object in level):
    │         if (object visible and in explored area):
    │             determine thing type and color
    │             transform position to screen
    │             draw marker (dot, arrow, or icon)
    │
    └─► Step 4: Draw player position
          draw arrow at player location
          arrow points in player facing direction

overhead_map_end():
    │
    ├─► Restore graphics state
    └─► Draw any overlays
```

---

## 20.8 Coordinate Transformation

```c
void world_to_overhead_map_point(
    world_point2d *world,
    point2d *screen,
    struct overhead_map_data *data)
{
    // Center on origin, apply scale
    screen->x = data->left + data->half_width +
                WORLD_TO_SCREEN(world->x, data->origin.x, data->scale);
    screen->y = data->top + data->half_height -  // Note: Y inverted
                WORLD_TO_SCREEN(world->y, data->origin.y, data->scale);
}
```

### Example

```
Player at world (4096, 6144), origin at (4096, 4096), scale 3:

X offset: (4096 - 4096) >> (8-3) = 0 >> 5 = 0
Y offset: (6144 - 4096) >> (8-3) = 2048 >> 5 = 64

Screen position: (left + half_width + 0, top + half_height - 64)
                = (100 + 64 + 0, 100 + 64 - 64)
                = (164, 100)

Player appears 64 pixels above center of map
```

---

## 20.9 Player Marker

```
Arrow showing player position and facing:

        ▲
       ╱│╲
      ╱ │ ╲
     ╱  │  ╲
    ╱   │   ╲

Arrow rotates based on player facing direction
Size scales with map zoom level
```

---

## 20.10 Summary

Marathon's automap system provides:

- **Compact exploration tracking** (~200 bytes per level)
- **Bitmask storage** for lines and polygons
- **4 zoom levels** for navigation flexibility
- **Color-coded display** for terrain and entities
- **Three rendering modes** for different contexts

### Key Source Files

| File | Purpose |
|------|---------|
| `overhead_map.c` | Automap rendering |
| `map.h` | Exploration bitmask macros |
| `overhead_map_macintosh.c` | Mac drawing primitives |

---

*This concludes the Marathon 2 Source Code Tutorial series.*
