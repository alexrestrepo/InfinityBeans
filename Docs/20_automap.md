# Chapter 20: Automap System

## Overhead Map and Exploration Tracking

> **Source files**: `overhead_map.c`, `overhead_map.h`, `overhead_map_macintosh.c`, `map.h`, `map.c`
> **Related chapters**: [Chapter 4: World](04_world.md), [Chapter 5: Rendering](05_rendering.md)

> **For Porting:** The automap logic in `overhead_map.c` is portable. Only the drawing primitives in `overhead_map_macintosh.c` need replacement—swap QuickDraw calls with your graphics library.

---

## 20.1 What Problem Are We Solving?

Marathon's levels are complex 3D environments. Players need:

- **Navigation aid** - Know where they've been and where to go
- **Exploration tracking** - Remember which areas are visited
- **Entity awareness** - See enemies and items on the map
- **Mission context** - Terminal checkpoints show objectives

**The constraints:**
- Memory efficient (levels can have 1000+ lines/polygons)
- Fast rendering during gameplay
- Different modes for different contexts (game, checkpoint, save preview)

---

## 20.2 Exploration Tracking (`map.h:843-855`, `map.c:95-96`)

Marathon tracks exploration with compact bitmask arrays:

### Global Variables (`map.c:95-96`)

```c
byte *automap_lines;
byte *automap_polygons;
```

### External Declarations (`map.h:843-844`)

```c
extern byte *automap_lines;
extern byte *automap_polygons;
```

### Access Macros (`map.h:851-855`)

```c
#define ADD_LINE_TO_AUTOMAP(i) (automap_lines[(i)>>3] |= (byte) 1<<((i)&0x07))
#define LINE_IS_IN_AUTOMAP(i) ((automap_lines[(i)>>3]&((byte)1<<((i)&0x07)))?(TRUE):(FALSE))

#define ADD_POLYGON_TO_AUTOMAP(i) (automap_polygons[(i)>>3] |= (byte) 1<<((i)&0x07))
#define POLYGON_IS_IN_AUTOMAP(i) ((automap_polygons[(i)>>3]&((byte)1<<((i)&0x07)))?(TRUE):(FALSE))
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

Lines and polygons are added to the automap during rendering:

- `render.c:879`: `ADD_POLYGON_TO_AUTOMAP(*polygon_index);` - when player enters polygon
- `render.c:977`: `ADD_LINE_TO_AUTOMAP(crossed_line_index)` - when line becomes visible
- `overhead_map.c:576-577`: During false automap generation for checkpoints

---

## 20.3 Overhead Map Data Structure (`overhead_map.h:17-28`)

```c
struct overhead_map_data
{
    short mode;                    /* Rendering mode (0-2) */
    short scale;                   /* Zoom level [1-4] */
    world_point2d origin;          /* Center of view in world coordinates */
    short origin_polygon_index;    /* Which polygon origin is in */
    short half_width, half_height; /* Half dimensions of map area */
    short width, height;           /* Full dimensions */
    short top, left;               /* Screen position offset */

    boolean draw_everything;       /* Debug mode - show all */
};
```

---

## 20.4 Rendering Modes (`overhead_map.h:10-15`)

```c
enum /* modes */
{
    _rendering_saved_game_preview,  /* 0 - Thumbnail in save dialog */
    _rendering_checkpoint_map,      /* 1 - Terminal checkpoint display */
    _rendering_game_map             /* 2 - Live gameplay map */
};
```

| Mode | Shows Explored Only | Shows Player | Shows Entities | Interactive |
|------|---------------------|--------------|----------------|-------------|
| `_rendering_game_map` | Yes | Yes | Yes | Yes |
| `_rendering_checkpoint_map` | Uses false automap | No | Checkpoint only | No |
| `_rendering_saved_game_preview` | Yes | Yes | No | No |

---

## 20.5 Scale System (`overhead_map.h:6-8`, `overhead_map.c:92-93`)

### Scale Constants (`overhead_map.h:6-8`)

```c
#define OVERHEAD_MAP_MINIMUM_SCALE 1   /* Zoomed out (overview) */
#define OVERHEAD_MAP_MAXIMUM_SCALE 4   /* Zoomed in (detail) */
#define DEFAULT_OVERHEAD_MAP_SCALE 3
```

### Coordinate Transformation (`overhead_map.c:92-93`)

```c
#define WORLD_TO_SCREEN_SCALE_ONE 8
#define WORLD_TO_SCREEN(x, x0, scale) (((x)-(x0))>>(WORLD_TO_SCREEN_SCALE_ONE-(scale)))
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

### Polygon Colors (`overhead_map.c:49-58`)

```c
enum /* polygon colors */
{
    _polygon_color,           /* 0 - Normal floor */
    _polygon_platform_color,  /* 1 - Elevator/door */
    _polygon_water_color,     /* 2 - Water media */
    _polygon_lava_color,      /* 3 - Lava media */
    _polygon_goo_color,       /* 4 - Goo media */
    _polygon_sewage_color,    /* 5 - Sewage media */
    _polygon_hill_color       /* 6 - KOTH hill */
};
```

### Polygon Color RGB Values (`overhead_map_macintosh.c:42-51`)

```c
static RGBColor polygon_colors[]=
{
    {0, 12000, 0},              /* _polygon_color - dark green */
    {30000, 0, 0},              /* _polygon_platform_color - red */
    {14*256, 37*256, 63*256},   /* _polygon_water_color - blue */
    {76*256, 27*256, 0},        /* _polygon_lava_color - orange */
    {137*256, 0, 137*256},      /* _polygon_goo_color - purple */
    {70*256, 90*256, 0},        /* _polygon_sewage_color - olive */
    {32768, 32768, 0}           /* _polygon_hill_color - yellow */
};
```

### Line Colors (`overhead_map.c:60-65`)

```c
enum /* line colors */
{
    _solid_line_color,         /* 0 - Impassable walls */
    _elevation_line_color,     /* 1 - Height change */
    _control_panel_line_color  /* 2 - Interactive panels */
};
```

### Line Definitions (`overhead_map_macintosh.c:91-96`)

```c
struct line_definition line_definitions[]=
{
    {{0, 65535, 0}, {1, 2, 2, 4}},    /* _solid_line_color - bright green */
    {{0, 40000, 0}, {1, 1, 1, 2}},    /* _elevation_line_color - medium green */
    {{65535, 0, 0}, {1, 2, 2, 4}}     /* _control_panel_line_color - red */
};
```

### Thing Colors (`overhead_map.c:67-75`)

```c
enum /* thing colors */
{
    _civilian_thing,    /* 0 - Friendly BOBs */
    _item_thing,        /* 1 - Pickups */
    _monster_thing,     /* 2 - Enemies */
    _projectile_thing,  /* 3 - Bullets/rockets */
    _checkpoint_thing,  /* 4 - Terminal checkpoint */
    NUMBER_OF_THINGS
};
```

### Thing Definitions (`overhead_map_macintosh.c:152-159`)

```c
struct thing_definition thing_definitions[NUMBER_OF_THINGS]=
{
    {{0, 0, 65535}, _rectangle_thing, {1, 2, 4, 8}},       /* civilian - blue */
    {{65535, 65535, 65535}, _rectangle_thing, {1, 2, 3, 4}}, /* item - white */
    {{65535, 0, 0}, _rectangle_thing, {1, 2, 4, 8}},       /* monster - red */
    {{65535, 65535, 0}, _rectangle_thing, {1, 1, 2, 3}},   /* projectile - yellow */
    {{65535, 0, 0}, _circle_thing, {8, 16, 16, 16}}        /* checkpoint - red circle */
};
```

### Thing Shapes (`overhead_map.c:77-81`)

```c
enum
{
    _rectangle_thing,  /* 0 - Square marker */
    _circle_thing      /* 1 - Circular marker */
};
```

---

## 20.7 Render Flags (`overhead_map.c:83-88`)

These flags mark which elements are visible on screen during the current frame:

```c
enum /* render flags */
{
    _endpoint_on_automap= 0x2000,  /* Endpoint is within screen bounds */
    _line_on_automap= 0x4000,      /* Line is visible */
    _polygon_on_automap= 0x8000    /* Polygon has visible endpoint */
};
```

---

## 20.8 Rendering Pipeline (`overhead_map.c:120-433`)

### Main Function (`overhead_map.c:120-433`)

```c
void _render_overhead_map(
    struct overhead_map_data *data)
{
    world_distance x0= data->origin.x, y0= data->origin.y;
    short scale= data->scale;
    world_point2d location;
    short i;

    if (data->mode==_rendering_checkpoint_map) generate_false_automap(data->origin_polygon_index);

    transform_endpoints_for_overhead_map(data);

    /* Step 1: shade all visible polygons (lines 146-200) */
    for (i=0;i<dynamic_world->polygon_count;++i)
    {
        struct polygon_data *polygon= get_polygon_data(i);
        if (POLYGON_IS_IN_AUTOMAP(i) && TEST_STATE_FLAG(i, _polygon_on_automap)
            &&(polygon->floor_transfer_mode!=_xfer_landscape||polygon->ceiling_transfer_mode!=_xfer_landscape))
        {
            /* Determine polygon color based on type and media */
            short color= _polygon_color;
            switch (polygon->type)
            {
                case _polygon_is_platform:
                    color= PLATFORM_IS_SECRET(get_platform_data(polygon->permutation)) ?
                        _polygon_color : _polygon_platform_color;
                    break;
            }

            if (polygon->media_index!=NONE)
            {
                struct media_data *media= get_media_data(polygon->media_index);
                if (media->height>=polygon->floor_height)
                {
                    switch (media->type)
                    {
                        case _media_water: color= _polygon_water_color; break;
                        case _media_lava: color= _polygon_lava_color; break;
                        case _media_goo: color= _polygon_goo_color; break;
                        case _media_jjaro: case _media_sewage: color= _polygon_sewage_color; break;
                    }
                }
            }

            draw_overhead_polygon(polygon->vertex_count, polygon->endpoint_indexes, color, scale);
        }
    }

    /* Step 2: draw all visible lines (lines 202-244) */
    for (i=0;i<dynamic_world->line_count;++i)
    {
        if (LINE_IS_IN_AUTOMAP(i))
        {
            /* Determine line color based on solid/elevation */
            /* ... */
            draw_overhead_line(i, line_color, scale);
        }
    }

    /* Step 3: print all visible annotations (lines 246-263) */
    /* Step 4: draw objects/things (lines 302-421) */
    /* Step 5: draw map name (line 423) */
}
```

### Rendering Flow

```
_render_overhead_map():
    │
    ├─► If checkpoint mode: generate_false_automap()
    │
    ├─► transform_endpoints_for_overhead_map()
    │     └─► Transform world coords to screen coords
    │     └─► Set _endpoint_on_automap, _polygon_on_automap flags
    │
    ├─► Step 1: Draw polygons (area fills)
    │     for (each polygon):
    │         if (POLYGON_IS_IN_AUTOMAP && on screen):
    │             determine color from type/media
    │             draw_overhead_polygon()
    │
    ├─► Step 2: Draw lines (edges)
    │     for (each line):
    │         if (LINE_IS_IN_AUTOMAP):
    │             determine line color
    │             draw_overhead_line()
    │
    ├─► Step 3: Draw annotations (text labels)
    │
    ├─► Step 4: Draw things (entities)
    │     ├─► Players: draw_overhead_player() - arrow shape
    │     ├─► Monsters: draw_overhead_thing() - rectangle
    │     ├─► Items: draw_overhead_thing() - rectangle
    │     └─► Checkpoint: draw_overhead_thing() - circle
    │
    └─► Step 5: draw_map_name() - level name at top
```

---

## 20.9 Endpoint Transformation (`overhead_map.c:437-485`)

```c
static void transform_endpoints_for_overhead_map(
    struct overhead_map_data *data)
{
    world_distance x0= data->origin.x, y0= data->origin.y;
    short screen_width= data->width, screen_height= data->height;
    short scale= data->scale;
    short i;

    /* transform all endpoints into screen space */
    for (i=0;i<dynamic_world->endpoint_count;++i)
    {
        struct endpoint_data *endpoint= get_endpoint_data(i);

        endpoint->transformed.x= data->half_width + WORLD_TO_SCREEN(endpoint->vertex.x, x0, scale);
        endpoint->transformed.y= data->half_height + WORLD_TO_SCREEN(endpoint->vertex.y, y0, scale);

        if (endpoint->transformed.x>=0 && endpoint->transformed.y>=0 &&
            endpoint->transformed.y<=screen_height && endpoint->transformed.x<=screen_width)
        {
            SET_STATE_FLAG(i, _endpoint_on_automap, TRUE);
        }
    }

    /* determine which polygons are visible based on endpoints */
    for (i=0;i<dynamic_world->polygon_count;++i)
    {
        struct polygon_data *polygon= get_polygon_data(i);
        short j;

        for (j=0;j<polygon->vertex_count;++j)
        {
            if (TEST_STATE_FLAG(polygon->endpoint_indexes[j], _endpoint_on_automap))
            {
                SET_STATE_FLAG(i, _polygon_on_automap, TRUE);
                break;
            }
        }
    }
}
```

### Transformation Example

```
Player at world (4096, 6144), origin at (4096, 4096), scale 3:

X offset: (4096 - 4096) >> (8-3) = 0 >> 5 = 0
Y offset: (6144 - 4096) >> (8-3) = 2048 >> 5 = 64

Screen position: (half_width + 0, half_height - 64)

Player appears 64 pixels above center of map
```

---

## 20.10 False Automap System (`overhead_map.c:491-580`)

For checkpoint maps in terminals, Marathon generates a "false" automap that shows the area around the checkpoint without requiring the player to have explored it.

### Generate False Automap (`overhead_map.c:491-519`)

```c
static void generate_false_automap(
    short polygon_index)
{
    long automap_line_buffer_size, automap_polygon_buffer_size;

    automap_line_buffer_size= (dynamic_world->line_count/8+((dynamic_world->line_count%8)?1:0))*sizeof(byte);
    automap_polygon_buffer_size= (dynamic_world->polygon_count/8+((dynamic_world->polygon_count%8)?1:0))*sizeof(byte);

    /* save current automap, then clear it */
    saved_automap_lines= (byte *) malloc(automap_line_buffer_size);
    saved_automap_polygons= (byte *) malloc(automap_polygon_buffer_size);

    if (saved_automap_lines && saved_automap_polygons)
    {
        memcpy(saved_automap_lines, automap_lines, automap_line_buffer_size);
        memcpy(saved_automap_polygons, automap_polygons, automap_polygon_buffer_size);
        memset(automap_lines, 0, automap_line_buffer_size);
        memset(automap_polygons, 0, automap_polygon_buffer_size);

        /* flood fill from checkpoint polygon */
        polygon_index= flood_map(polygon_index, LONG_MAX, false_automap_cost_proc, _breadth_first, (void *) NULL);
        do
        {
            polygon_index= flood_map(NONE, LONG_MAX, false_automap_cost_proc, _breadth_first, (void *) NULL);
        }
        while (polygon_index!=NONE);
    }
}
```

### Replace Real Automap (`overhead_map.c:521-542`)

```c
static void replace_real_automap(
    void)
{
    if (saved_automap_lines)
    {
        long automap_line_buffer_size= (dynamic_world->line_count/8+((dynamic_world->line_count%8)?1:0))*sizeof(byte);
        memcpy(automap_lines, saved_automap_lines, automap_line_buffer_size);
        free(saved_automap_lines);
        saved_automap_lines= (byte *) NULL;
    }

    if (saved_automap_polygons)
    {
        long automap_polygon_buffer_size= (dynamic_world->polygon_count/8+((dynamic_world->polygon_count%8)?1:0))*sizeof(byte);
        memcpy(automap_polygons, saved_automap_polygons, automap_polygon_buffer_size);
        free(saved_automap_polygons);
        saved_automap_polygons= (byte *) NULL;
    }
}
```

---

## 20.11 Player Arrow Rendering (`overhead_map_macintosh.c:213-258`)

```c
static void draw_overhead_player(
    world_point2d *center,
    angle facing,
    short color,
    short scale)
{
    world_point2d triangle[3];
    struct entity_definition *definition;

    definition= entity_definitions+color;

    /* Build triangle pointing in facing direction */
    triangle[0]= triangle[1]= triangle[2]= *center;
    translate_point2d(triangle+0, definition->front>>(OVERHEAD_MAP_MAXIMUM_SCALE-scale), facing);
    translate_point2d(triangle+1, definition->rear>>(OVERHEAD_MAP_MAXIMUM_SCALE-scale),
        normalize_angle(facing+definition->rear_theta));
    translate_point2d(triangle+2, definition->rear>>(OVERHEAD_MAP_MAXIMUM_SCALE-scale),
        normalize_angle(facing-definition->rear_theta));

    if (scale < 2)
    {
        /* At low zoom, draw as line */
        MoveTo(triangle[0].x, triangle[0].y);
        LineTo(triangle[1].x, triangle[1].y);
    }
    else
    {
        /* At high zoom, draw as filled triangle */
        polygon= OpenPoly();
        MoveTo(triangle[2].x, triangle[2].y);
        for (i=0;i<3;++i) LineTo(triangle[i].x, triangle[i].y);
        ClosePoly();
        FillPoly(polygon, &qd.black);
        KillPoly(polygon);
    }
}
```

### Player Arrow Visualization

```
Arrow showing player position and facing:

        ▲
       ╱│╲
      ╱ │ ╲
     ╱  │  ╲
    ╱   │   ╲

Arrow rotates based on player facing direction
Size scales with map zoom level
front = 16, rear = 10, rear_theta = 7*512/20 = 179.2°
```

---

## 20.12 Summary

Marathon's automap system provides:

- **Compact exploration tracking** (~200 bytes per level)
- **Bitmask storage** for lines and polygons (`map.h:851-855`)
- **4 zoom levels** for navigation flexibility (`overhead_map.h:6-8`)
- **Color-coded display** for terrain and entities (`overhead_map.c:49-75`)
- **Three rendering modes** for different contexts (`overhead_map.h:10-15`)
- **False automap generation** for checkpoint maps (`overhead_map.c:491-519`)

### Key Constants

| Constant | Value | Source |
|----------|-------|--------|
| `OVERHEAD_MAP_MINIMUM_SCALE` | 1 | `overhead_map.h:6` |
| `OVERHEAD_MAP_MAXIMUM_SCALE` | 4 | `overhead_map.h:7` |
| `DEFAULT_OVERHEAD_MAP_SCALE` | 3 | `overhead_map.h:8` |
| `WORLD_TO_SCREEN_SCALE_ONE` | 8 | `overhead_map.c:92` |
| `_endpoint_on_automap` | 0x2000 | `overhead_map.c:85` |
| `_polygon_on_automap` | 0x8000 | `overhead_map.c:87` |
| `NUMBER_OF_THINGS` | 5 | `overhead_map.c:74` |

### Key Source Files

| File | Purpose |
|------|---------|
| `overhead_map.c` | Automap rendering logic (581 lines) |
| `overhead_map.h` | Data structure, modes, scale constants |
| `overhead_map_macintosh.c` | Mac drawing primitives (304 lines) |
| `map.h` | Exploration bitmask macros (lines 843-855) |
| `map.c` | automap_lines/polygons globals (lines 95-96) |
| `render.c` | Exploration triggers (lines 879, 977) |

---

## 20.13 See Also

- [Chapter 4: World](04_world.md) — Polygon and line data structures
- [Chapter 5: Rendering](05_rendering.md) — Where exploration is triggered
- [Chapter 28: Terminals](28_terminals.md) — Checkpoint map display

---

*Next: [Chapter 21: HUD and Interface](21_hud.md) - Heads-up display and status rendering*
