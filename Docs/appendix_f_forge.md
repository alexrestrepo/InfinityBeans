# Appendix F: Map Creation Constraints (Forge Reference)

## What a Map Editor Needs to Know

> **Note**: Bungie's Forge editor source is not publicly available. This appendix documents the constraints that the Marathon engine expects from map data, derived from the engine source code. Map editors must produce data satisfying these constraints for the engine to function correctly.

---

## F.1 Fundamental Limits

The engine enforces hard limits on map geometry:

```c
// From map.h:25-28, 532
#define MAXIMUM_POLYGONS_PER_MAP    1024     // KILO
#define MAXIMUM_SIDES_PER_MAP       4096     // 4*KILO
#define MAXIMUM_ENDPOINTS_PER_MAP   8192     // 8*KILO
#define MAXIMUM_LINES_PER_MAP       4096     // 4*KILO
#define MAXIMUM_VERTICES_PER_POLYGON  8      // Per-polygon vertex limit
```

**Practical Implications:**

| Limit | Value | Notes |
|-------|-------|-------|
| Polygons | 1,024 | Rooms/areas in the level |
| Endpoints | 8,192 | 2D vertices (corner points) |
| Lines | 4,096 | Connections between endpoints |
| Sides | 4,096 | Textured surfaces on lines |
| Vertices per polygon | 8 | Maximum polygon complexity |

> **Design Note:** These limits were chosen for 1995 hardware constraints. 1024 polygons at 30 ticks/sec with flood-fill pathfinding and portal rendering was the practical maximum for a 68040 Mac.

---

## F.2 Polygon Construction Rules

### Clockwise Winding Order

Polygon vertices **must** be stored in clockwise order when viewed from above:

```c
// From map.h:578
short endpoint_indexes[MAXIMUM_VERTICES_PER_POLYGON]; /* clockwise */
```

The engine calculates clockwise order automatically from line ownership:

```c
// From map_constructors.c:374-388
/* given a polygon, return its endpoints in clockwise order */
static short calculate_clockwise_endpoints(short polygon_index, short *buffer)
{
    struct polygon_data *polygon = get_polygon_data(polygon_index);
    for (i = 0; i < polygon->vertex_count; ++i)
    {
        *buffer++ = clockwise_endpoint_in_line(polygon_index, polygon->line_indexes[i], 0);
    }
    return polygon->vertex_count;
}
```

### Why Clockwise Matters

The clockwise/counterclockwise distinction determines:
1. **Which side of a line faces which polygon**
2. **Texture mapping direction**
3. **Normal direction for collision detection**

```
Top-Down View (Y increases downward):

    Polygon A                    Polygon B
         ●───────────────────────────●
        e0          Line           e1

Clockwise polygon owner:    Looking from e0 to e1, polygon is on RIGHT
Counterclockwise owner:     Looking from e0 to e1, polygon is on LEFT

Cross-product test (from map_constructors.c:800):
    cross = (e1.x - e0.x) * (center.y - e1.y) - (e1.y - e0.y) * (center.x - e1.x)
    if (cross > 0): clockwise
    if (cross < 0): counterclockwise
```

### Convexity Requirement

While not explicitly stated, the texture mapper in `scottish_textures.c` assumes convex polygons:

```c
// From scottish_textures.c:54
/* for non-convex or otherwise weird lines (dx<=0, dy<=0) we don't draw anything */
```

The renderer searches for vertices in clockwise and counterclockwise order assuming the polygon is convex:

```c
// From scottish_textures.c:122
/* clockwise vertices for this convex polygon */
```

**Editor Constraint:** All polygons should be convex. Concave polygons may render incorrectly or not at all.

---

## F.3 Line-Polygon Ownership

Each line can be owned by at most two polygons—one on each side:

```c
// From map.h:424-430
struct line_data {
    /* a line can be owned by a clockwise polygon, a counterclockwise polygon,
       or both (but never two of the same) (can be NONE) */
    short clockwise_polygon_owner;
    short counterclockwise_polygon_owner;

    /* the side definition facing each owner (can be NONE) */
    short clockwise_polygon_side_index;
    short counterclockwise_polygon_side_index;
};
```

### Line Types

| Scenario | clockwise_owner | counterclockwise_owner | Result |
|----------|-----------------|------------------------|--------|
| Interior wall | Polygon A | Polygon B | Portal between A and B |
| Exterior wall | Polygon A | NONE | Solid impassable wall |
| Detached | NONE | NONE | Invalid/unused line |

### Adjacent Polygon Calculation

When loading a map, the engine calculates which polygons are adjacent:

```c
// From map_constructors.c:419-446
static void calculate_adjacent_polygons(short polygon_index, short *polygon_indexes)
{
    struct polygon_data *polygon = get_polygon_data(polygon_index);
    for (i = 0; i < polygon->vertex_count; ++i)
    {
        struct line_data *line = get_line_data(polygon->line_indexes[i]);

        if (polygon_index == line->clockwise_polygon_owner)
        {
            adjacent_polygon_index = line->counterclockwise_polygon_owner;
        }
        else
        {
            assert(polygon_index == line->counterclockwise_polygon_owner);
            adjacent_polygon_index = line->clockwise_polygon_owner;
        }
        *polygon_indexes++ = adjacent_polygon_index;  // Can be NONE
    }
}
```

---

## F.4 Side (Wall Texture) Definitions

Sides define how textures are applied to lines. Each line can have 0, 1, or 2 sides:

### Side Types

```c
// From map.h:478-486
enum /* side types */
{
    _full_side,      // Floor-to-ceiling texture
    _high_side,      // Panel from ceiling down (step up)
    _low_side,       // Panel from floor up (step down)
    _composite_side, // Full texture with overlay (control panel)
    _split_side      // Ceiling panel + floor panel (window)
};
```

**Visual Guide:**

```
_full_side:           _high_side:         _low_side:          _split_side:
┌──────────────┐      ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│              │      │   PRIMARY    │    │              │    │   PRIMARY    │
│   PRIMARY    │      │   texture    │    │              │    │   texture    │
│   texture    │      ├──────────────┤    ├──────────────┤    ├──────────────┤
│              │      │   (open      │    │   PRIMARY    │    │              │
│              │      │    portal)   │    │   texture    │    │   SECONDARY  │
└──────────────┘      └──────────────┘    └──────────────┘    └──────────────┘
```

### Texture Coordinate Origins

Each side has texture origin offsets:

```c
// From map.h:488-492
struct side_texture_definition
{
    world_distance x0, y0;  // Texture offset from polygon origin
    shape_descriptor texture;
};
```

The engine calculates exclusion zones for collision:

```c
// From map_constructors.c:264-266
side->exclusion_zone.e0 = side->exclusion_zone.e2 = *e0;
side->exclusion_zone.e1 = side->exclusion_zone.e3 = *e1;
push_out_line(&side->exclusion_zone.e0, &side->exclusion_zone.e1,
              MINIMUM_SEPARATION_FROM_WALL, line->length);
```

---

## F.5 Polygon Types

Polygons can have special behaviors:

```c
// From map.h:534-555
enum /* polygon types */
{
    _polygon_is_normal,                    // Standard walkable area
    _polygon_is_item_impassable,           // Items can't enter
    _polygon_is_monster_impassable,        // Monsters can't enter
    _polygon_is_hill,                      // King-of-the-hill zone
    _polygon_is_base,                      // Team base (CTF)
    _polygon_is_platform,                  // Moving platform
    _polygon_is_light_on_trigger,          // Trigger to turn on light
    _polygon_is_platform_on_trigger,       // Trigger to activate platform
    _polygon_is_light_off_trigger,         // Trigger to turn off light
    _polygon_is_platform_off_trigger,      // Trigger to deactivate platform
    _polygon_is_teleporter,                // Teleport destination in .permutation
    _polygon_is_zone_border,               // AI zone boundary
    _polygon_is_goal,                      // Mission objective
    _polygon_is_visible_monster_trigger,   // Activate visible monsters
    _polygon_is_invisible_monster_trigger, // Activate invisible monsters
    _polygon_is_dual_monster_trigger,      // Activate both types
    _polygon_is_item_trigger,              // Activate items in zone
    _polygon_must_be_explored,             // Required for level completion
    _polygon_is_automatic_exit             // Auto-complete level
};
```

### The Permutation Field

Many polygon types use the `permutation` field to store associated data:

| Polygon Type | permutation Contains |
|--------------|---------------------|
| `_polygon_is_platform` | Platform index |
| `_polygon_is_teleporter` | Destination polygon index |
| `_polygon_is_light_on_trigger` | Light source index |
| `_polygon_is_base` | Team number |
| `_polygon_is_goal` | Goal number |

---

## F.6 Control Panels (Switches)

Sides can have interactive control panels:

```c
// From map.h:450-461
enum /* control panel side types */
{
    _panel_is_oxygen_refuel,
    _panel_is_shield_refuel,
    _panel_is_double_shield_refuel,
    _panel_is_triple_shield_refuel,
    _panel_is_light_switch,        // permutation = light index
    _panel_is_platform_switch,     // permutation = platform index
    _panel_is_tag_switch,          // permutation = tag (NONE = tagless)
    _panel_is_pattern_buffer,      // Save game terminal
    _panel_is_computer_terminal,   // Story terminal
    NUMBER_OF_CONTROL_PANELS
};
```

### Control Panel Flags

```c
// From map.h:437-448
enum /* side flags */
{
    _control_panel_status = 0x0001,                // On/off state
    _side_is_control_panel = 0x0002,               // Is a control panel
    _side_is_repair_switch = 0x0004,               // Must toggle to exit
    _side_is_destructive_switch = 0x0008,          // Uses an item
    _side_is_lighted_switch = 0x0010,              // Must be lit to use
    _side_switch_can_be_destroyed = 0x0020,        // Projectile can toggle
    _side_switch_can_only_be_hit_by_projectiles = 0x0040
};
```

---

## F.7 Transfer Modes (Special Effects)

Transfer modes control how textures are rendered:

```c
// From map.h:303-327
enum /* object transfer modes */
{
    _xfer_normal,              // Standard texture
    _xfer_fade_out_to_black,   // Fading out
    _xfer_invisibility,        // Invisible (predator-style)
    _xfer_subtle_invisibility, // Partially visible
    _xfer_pulsate,             // Brightness pulsing (polygons only)
    _xfer_wobble,              // Texture distortion (polygons only)
    _xfer_fast_wobble,         // Fast distortion
    _xfer_static,              // TV static noise
    _xfer_50percent_static,    // Partial static
    _xfer_landscape,           // Parallax background
    _xfer_smear,               // Repeat pixel(0,0) everywhere
    _xfer_fade_out_static,     // Static fading out
    _xfer_pulsating_static,    // Pulsing static
    _xfer_fold_in,             // Appear (teleport in)
    _xfer_fold_out,            // Disappear (teleport out)
    _xfer_horizontal_slide,    // Scrolling horizontal
    _xfer_fast_horizontal_slide,
    _xfer_vertical_slide,      // Scrolling vertical
    _xfer_fast_vertical_slide,
    _xfer_wander,              // Random movement
    _xfer_fast_wander,
    _xfer_big_landscape        // Large parallax
};
```

---

## F.8 Height Constraints

Floor and ceiling heights must satisfy certain constraints:

### Line Height Calculation

```c
// From map_constructors.c:174-202
if (polygon1 && polygon2)
{
    line->highest_adjacent_floor = MAX(polygon1->floor_height, polygon2->floor_height);
    line->lowest_adjacent_ceiling = MIN(polygon1->ceiling_height, polygon2->ceiling_height);
    if (polygon1->floor_height != polygon2->floor_height) elevation = TRUE;
}
```

### Passability Rules

```
Solid line (wall):
    Line connects only one polygon (exterior wall)
    OR line explicitly flagged as solid

Passable portal:
    ceiling - floor >= player_height  (at minimum WORLD_ONE * 4/5 = ~819 units)
    Step up height <= WORLD_ONE/3 (~341 units)
```

### Endpoint Height Precalculation

```c
// From map_constructors.c:99-100
world_distance highest_adjacent_floor_height = SHORT_MIN;
world_distance lowest_adjacent_ceiling_height = SHORT_MAX;
```

The engine iterates all lines touching an endpoint to find the highest floor and lowest ceiling, used for collision detection.

---

## F.9 Detached Polygons

Polygons can be "detached" (not connected to the main level geometry):

```c
// From map.h:557-559
#define POLYGON_IS_DETACHED_BIT 0x4000
#define POLYGON_IS_DETACHED(p) ((p)->flags & POLYGON_IS_DETACHED_BIT)
```

From the developer notes:

```c
// From map_constructors.c:18-19
/* detached polygons (i.e., shadows) and their twins will not have their
   neighbor polygon lists correctly computed */
```

**Use cases:**
- Shadow polygons for visual effects
- Decals or overlays
- Special rendering areas

**Constraints:**
- Detached polygons don't participate in collision
- AI won't pathfind through detached polygons
- Must have a "twin" polygon for proper behavior

---

## F.10 Map Validation Checklist

Before saving a map, editors should verify:

| Check | Constraint | Source |
|-------|------------|--------|
| Polygon count | ≤ 1024 | map.h:25 |
| Vertices per polygon | ≤ 8 | map.h:532 |
| Line count | ≤ 4096 | map.h:28 |
| Side count | ≤ 4096 | map.h:26 |
| Endpoint count | ≤ 8192 | map.h:27 |
| Winding order | Clockwise | map.h:578 |
| Convexity | All polygons convex | scottish_textures.c:54 |
| Line ownership | Max 2 polygons per line | map.h:428-430 |
| Height validity | ceiling > floor | Implicit |
| Portal clearance | ≥ player height | physics.c |

---

## F.11 Exclusion Zone Calculation

At load time, the engine precalculates collision data:

```c
// From map_constructors.c:476-547
void precalculate_map_indexes(void)
{
    for (polygon_index = 0; polygon_index < dynamic_world->polygon_count; ++polygon_index)
    {
        // Find nearby lines for collision
        find_intersecting_endpoints_and_lines(polygon_index, MINIMUM_SEPARATION_FROM_WALL,
            line_indexes, &line_count, endpoint_indexes, &endpoint_count,
            polygon_indexes, &polygon_count);

        // Store exclusion zone indices
        polygon->first_exclusion_zone_index = dynamic_world->map_index_count;
        polygon->line_exclusion_zone_count = ...;
        polygon->point_exclusion_zone_count = ...;
    }
}
```

This precalculation enables fast runtime collision detection by limiting which lines/endpoints each polygon needs to check.

---

## F.12 Summary

Key constraints for map creation:

1. **Polygons are convex** with clockwise vertex ordering
2. **Lines connect exactly 1-2 polygons** (exterior walls or portals)
3. **Height differences** determine passability and side types
4. **Transfer modes** apply per-surface visual effects
5. **Polygon types** control gameplay behavior
6. **Control panels** link to lights, platforms, or terminals
7. **Precalculated data** (areas, centers, exclusion zones) is generated at load time

The engine handles most redundant data calculation automatically via `recalculate_redundant_*` functions in `map_constructors.c`. Editors need only provide the core geometry and properties.

---

*Return to: [Appendix E: M2 vs Infinity](appendix_e_m2_vs_infinity.md) | [Table of Contents](README.md)*
