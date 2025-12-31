# Chapter 4: World Representation

## Building the Marathon Universe

> **For Porting:** `map.c`, `map.h`, and `map_constructors.c` are fully portable! All data structures use platform-independent types (`short`, `long`, `word`). Just ensure `word` is defined as `uint16_t` (see Chapter 24 for full type mappings) and handle byte swapping when loading from big-endian files.

---

## 4.1 What Problem Are We Solving?

Marathon needs to represent a 3D world that players can explore. This world consists of:
- **Rooms** with floors and ceilings at different heights
- **Walls** separating rooms, some solid, some you can see or walk through
- **Connections** between rooms (doorways, corridors, windows)
- **Dynamic elements** like doors, elevators, and water

**The constraints:**
- Must support efficient traversal (for rendering and AI pathfinding)
- Must handle complex indoor layouts with overlapping areas
- Must allow dynamic changes (doors opening, platforms moving)
- Must work with 1995 hardware and memory limits

**Why not use a simple grid?**

A grid (like Wolfenstein 3D) limits level design—every wall must be axis-aligned, every room rectangular. Marathon's designers wanted curved corridors, irregular rooms, and multi-level spaces.

**Why not use a BSP tree (like Doom)?**

BSP trees are great for rendering but harder to modify dynamically. Marathon needed moving platforms, doors, and crushers that change geometry every frame.

**Marathon's solution: Explicit Polygon Connectivity**

Marathon stores the world as a graph of connected polygons. Each polygon knows its neighbors, so traversal is O(1)—just follow the connection. This makes both rendering (portal culling) and physics (collision detection) efficient.

---

## 4.2 Understanding the Building Blocks

Before diving into code, let's understand how Marathon represents the world conceptually.

### The Four Levels of Geometry

Marathon's world is built from four types of elements, each referencing the ones below:

```
Level 4: POLYGONS (Rooms)
         ↓ reference
Level 3: LINES (Edges between rooms)
         ↓ reference
Level 2: SIDES (Wall textures and properties)
         ↓ reference
Level 1: ENDPOINTS (2D vertices)
```

Think of it like building a house:
- **Endpoints** are the corners where walls meet (just X,Y coordinates)
- **Lines** are the walls connecting corners
- **Sides** are the paint and wallpaper on each side of a wall
- **Polygons** are the rooms formed by connecting walls

### Visual Overview

```
A simple two-room level:

         e0 ────────L0──────── e1
         │                      │
         │                      │
        L3      Room A         L1
         │     (Polygon 0)      │
         │                      │
         │                      │
         e3 ═══════L2═══════ e2 ← This line is shared!
         │      (portal)        │
         │                      │
        L6      Room B         L4
         │     (Polygon 1)      │
         │                      │
         │                      │
         e5 ────────L5──────── e4

Key insight: Line L2 is referenced by BOTH polygons.
- When you're in Room A and cross L2, you enter Room B
- When you're in Room B and cross L2, you enter Room A
```

---

## 4.3 Let's Build: A Simple Version

Before seeing Marathon's actual code, let's build a simplified version to understand the concepts.

### Step 1: Define the Basic Structures

```c
// Simplified world representation

// A 2D point (vertex)
typedef struct {
    int x, y;  // World coordinates
} Point2D;

// A line connecting two points
typedef struct {
    int endpoint_a;     // Index into points array
    int endpoint_b;     // Index into points array
    int polygon_front;  // Polygon on "front" side (-1 if solid wall)
    int polygon_back;   // Polygon on "back" side (-1 if solid wall)
    bool is_solid;      // Can you walk through?
} Line;

// A convex polygon (room)
typedef struct {
    int vertex_count;
    int vertices[8];      // Indices into points array
    int lines[8];         // Indices into lines array
    int neighbors[8];     // Adjacent polygon for each line (-1 if wall)
    int floor_height;     // Z coordinate of floor
    int ceiling_height;   // Z coordinate of ceiling
} Polygon;

// The complete world
typedef struct {
    Point2D points[MAX_POINTS];
    Line lines[MAX_LINES];
    Polygon polygons[MAX_POLYGONS];
    int point_count, line_count, polygon_count;
} World;
```

### Step 2: Finding Which Room You're In

A key operation is determining which polygon contains a point:

```c
// Check if point P is inside polygon (2D test)
bool point_in_polygon(World* world, int polygon_index, int px, int py) {
    Polygon* poly = &world->polygons[polygon_index];
    int crossings = 0;

    for (int i = 0; i < poly->vertex_count; i++) {
        Point2D* v1 = &world->points[poly->vertices[i]];
        Point2D* v2 = &world->points[poly->vertices[(i + 1) % poly->vertex_count]];

        // Ray casting algorithm: count how many edges cross a ray
        // going from (px, py) to the right
        if ((v1->y <= py && v2->y > py) || (v2->y <= py && v1->y > py)) {
            float x_cross = v1->x + (float)(py - v1->y) / (v2->y - v1->y) * (v2->x - v1->x);
            if (px < x_cross) {
                crossings++;
            }
        }
    }

    return (crossings % 2) == 1;  // Odd = inside
}

// Find which polygon contains a point
int find_polygon_containing_point(World* world, int x, int y) {
    for (int i = 0; i < world->polygon_count; i++) {
        if (point_in_polygon(world, i, x, y)) {
            return i;
        }
    }
    return -1;  // Not in any polygon
}
```

### Step 3: Moving Between Rooms

When you cross a line, how do you know which room you've entered?

```c
// Find the polygon on the other side of a line
int find_adjacent_polygon(World* world, int current_polygon, int line_index) {
    Line* line = &world->lines[line_index];

    // If we're on the front side, return the back side (and vice versa)
    if (current_polygon == line->polygon_front) {
        return line->polygon_back;  // Could be -1 if solid wall
    } else {
        return line->polygon_front;
    }
}

// This is O(1) - no searching needed!
```

**Marathon's approach:** This is essentially what Marathon does, but with more sophisticated tracking of clockwise vs counterclockwise ownership to handle line direction consistently.

---

## 4.4 Marathon's World Data Structures

Now let's see the actual Marathon implementation. The structures are more complex because they support textures, lighting, platforms, and other features.

### The Base Point Type

Before looking at endpoints, here's the fundamental 2D point structure used throughout Marathon:

```c
struct world_point2d {  // 4 bytes
    world_distance x, y;  // Each is a short (16-bit signed)
};
```

This compact structure represents positions in Marathon's 2D world space. `world_distance` is just a `short` (16-bit), giving Marathon a coordinate range of ±32,767 world units.

### Endpoints (Vertices)

Marathon stores vertices as **endpoints** with additional metadata for rendering optimization:

```c
// From map.h:378 - 16 bytes per endpoint
struct endpoint_data {
    word flags;
    world_distance highest_adjacent_floor_height;   // For rendering optimization
    world_distance lowest_adjacent_ceiling_height;
    world_point2d vertex;       // The actual X,Y position
    world_point2d transformed;  // View-space coords (cached during render)
    short supporting_polygon_index;  // Which polygon "owns" this endpoint
};
```

**Why the cached heights?** When rendering, Marathon needs to know the height range visible at each vertex. Pre-computing these saves time during the render loop.

### Lines (Edges)

Lines connect endpoints and track which polygons they separate:

```c
// From map.h:416 - 32 bytes per line
struct line_data {
    short endpoint_indexes[2];             // The two endpoints
    word flags;                            // Solid, transparent, etc.

    world_distance length;                 // Precomputed (Pythagorean)
    world_distance highest_adjacent_floor;
    world_distance lowest_adjacent_ceiling;

    // The KEY ownership fields:
    short clockwise_polygon_owner;         // Polygon on CW side
    short counterclockwise_polygon_owner;  // Polygon on CCW side

    // Side indices (for textures on each side)
    short clockwise_polygon_side_index;
    short counterclockwise_polygon_side_index;
};
```

**Line Flags** (stored in `line_data.flags`):

| Flag | Value | Description |
|------|-------|-------------|
| `SOLID_LINE_BIT` | 0x4000 | Blocks movement (impassable) |
| `TRANSPARENT_LINE_BIT` | 0x2000 | Can see through (portal) |
| `LANDSCAPE_LINE_BIT` | 0x1000 | Uses sky texture |
| `ELEVATION_LINE_BIT` | 0x800 | Has height change |
| `VARIABLE_ELEVATION_LINE_BIT` | 0x400 | Platform edge |
| `LINE_HAS_TRANSPARENT_SIDE_BIT` | 0x200 | Has window texture |

**Usage Macros**: Marathon provides convenience macros for checking line properties:
- `LINE_IS_SOLID(l)` - Check if line blocks movement
- `LINE_IS_TRANSPARENT(l)` - Check if line is a portal
- `LINE_IS_LANDSCAPED(l)` - Check if line uses sky texture
- `LINE_HAS_TRANSPARENT_SIDE(l)` - Check if line has window texture

> **Source:** `map.h:416` for `struct line_data`, `map.h:391-413` for line flags

### The Clockwise/Counterclockwise System

This is Marathon's clever solution for consistent line ownership:

```
Understanding polygon ownership (which side of a line faces which polygon):

        Top view of two adjacent polygons sharing Line L:

                   e0 ─────────── e1
                    │             │
                    │  Polygon A  │
                    │             │
              e3 ── e4 ═══════════ e2 ── e5
                    │  (Line L)   │
                    │  Polygon B  │
                    │             │
                   e6 ─────────── e7

        Line L connects endpoints e4 and e2. Both polygons A and B
        reference Line L in their edge lists, but in OPPOSITE directions:

        Walking around Polygon A CLOCKWISE:
             e0 → e1 → e2 → e4 → e0
                        ↓
                   When traversing this edge,
                   Line L goes from e2 to e4
                   ∴ Polygon A is the CLOCKWISE owner

        Walking around Polygon B CLOCKWISE:
             e4 → e2 → e5 → e7 → e6 → e3 → e4
              ↓
         When traversing this edge,
         Line L goes from e4 to e2 (opposite direction!)
         ∴ Polygon B is the COUNTERCLOCKWISE owner

        The KEY insight: The same physical line is traversed in
        OPPOSITE directions by adjacent polygons. This determines
        which polygon is the clockwise vs counterclockwise owner.
```

### Detailed Connectivity: All Four Levels Together

To solidify this concept, here's how all four levels of Marathon's geometry connect:

```
LEVEL 1: ENDPOINTS (Vertices)
        Marathon stores 2D vertices as "endpoints" with metadata:

        Endpoint 0          Endpoint 1          Endpoint 2
        (x=100, y=200)      (x=300, y=200)      (x=300, y=400)
             e0                  e1                  e2
             *                   *                   *

LEVEL 2: LINES (Edges)
        Lines connect endpoints and store which polygons they separate:

        Line 0:                                Line 1:
        endpoint[0] = e0                       endpoint[0] = e1
        endpoint[1] = e1                       endpoint[1] = e2
        clockwise_owner = Polygon A            clockwise_owner = Polygon A
        counterclockwise_owner = Polygon B     counterclockwise_owner = Polygon B

             e0 ──────Line 0────── e1
                                   │
                                   │
                                Line 1
                                   │
                                   │
                                   e2

LEVEL 3: SIDES (Wall Surfaces)
        Each line can have TWO sides (one for each polygon):

        Line 0's sides:
        - Clockwise side (faces Polygon A)
          ├─ primary_texture = "Metal Wall"
          ├─ lightsource_index = 5
          └─ control_panel_type = NONE

        - Counterclockwise side (faces Polygon B)
          ├─ primary_texture = "Brick Wall"
          ├─ lightsource_index = 3
          └─ control_panel_type = SWITCH

        Visual representation:
                    Polygon A
                    ┌─────────────────┐
                    │                 │
                    │   "Metal Wall"  │  ← Clockwise side
        ────────────┼─────────────────┼────────────
                    │  "Brick Wall"   │  ← Counterclockwise side
                    │                 │
                    │    Polygon B    │
                    └─────────────────┘

LEVEL 4: POLYGONS (Rooms)
        Polygons are the final level, storing:
        - vertex_count = 4
        - endpoint_indexes[] = {e0, e1, e2, e3}
        - line_indexes[] = {Line0, Line1, Line2, Line3}
        - adjacent_polygon_indexes[] = {PolyB, PolyC, NONE, PolyD}
        - floor_height = 0
        - ceiling_height = 1024

        Complete polygon example:

             e0 ──────Line 0────── e1
             │                      │
             │                      │
          Line 3   Polygon A     Line 1
             │    (ceiling=1024)    │
             │    (floor=0)         │
             │                      │
             e3 ──────Line 2────── e2

        Adjacent polygon lookup:
        - Along Line 0: Polygon B
        - Along Line 1: Polygon C
        - Along Line 2: NONE (solid wall, no neighbor)
        - Along Line 3: Polygon D
```

**Adjacent Polygon Lookup** - O(1):
```c
// From map.c - find polygon on other side of a line
short find_adjacent_polygon(short polygon_index, short line_index)
{
    struct line_data *line = get_line_data(line_index);
    return (polygon_index == line->clockwise_polygon_owner) ?
           line->counterclockwise_polygon_owner :
           line->clockwise_polygon_owner;
}
```

### Sides (Wall Surfaces)

Each side of a line can have textures and properties:

```c
// From map.h:499 - 64 bytes per side
struct side_data {
    word type;   // _full_side, _high_side, _low_side, _split_side
    word flags;

    // Texture definitions for different parts of the wall
    struct side_texture_definition {
        world_distance x0, y0;       // Texture offset
        shape_descriptor texture;     // Which texture to use
    } primary_texture,    // Main wall texture
      secondary_texture,  // Upper/lower texture (for partial walls)
      transparent_texture; // Glass/window texture

    // Collision data
    world_point2d *exclusion_zone;  // Reserved

    // Panel info (for switches, terminals)
    short control_panel_type;
    short control_panel_permutation;

    // Lighting
    short primary_lightsource_index;
    short secondary_lightsource_index;
    short transparent_lightsource_index;

    fixed ambient_delta;  // Light adjustment
};
```

**Side Types:**
- `_full_side` - Floor to ceiling wall
- `_high_side` - Upper portion only (over a window)
- `_low_side` - Lower portion only (under a window)
- `_split_side` - Both upper and lower portions

> **Source:** `map.h:499` for `struct side_data`

### Polygons (Rooms)

The main room structure:

```c
// From map.h:571 - 128 bytes per polygon
struct polygon_data {
    word type;              // Normal, platform, teleporter, etc.
    word flags;
    word permutation;       // Type-specific data

    short vertex_count;     // Number of vertices (max 8)

    // These arrays define the polygon shape:
    short endpoint_indexes[MAXIMUM_VERTICES_PER_POLYGON];  // 8
    short line_indexes[MAXIMUM_VERTICES_PER_POLYGON];      // 8

    // Connectivity - which polygon is through each line:
    short adjacent_polygon_indexes[MAXIMUM_VERTICES_PER_POLYGON];

    // Geometry
    world_distance floor_height;
    world_distance ceiling_height;

    // Textures
    shape_descriptor floor_texture;
    shape_descriptor ceiling_texture;
    world_distance floor_origin_x, floor_origin_y;
    world_distance ceiling_origin_x, ceiling_origin_y;

    // Lighting
    short floor_lightsource_index;
    short ceiling_lightsource_index;

    // Environment
    short media_index;      // Water/lava/etc. (-1 if none)
    short media_lightsource_index;
    short ambient_sound_index;
    short random_sound_index;

    // Objects
    short first_object;     // Head of linked list of objects in this polygon
    short first_exclusion_zone_index;

    // Precomputed
    long area;              // For random point selection

    // For dynamic platforms
    short floor_transfer_mode;
    short ceiling_transfer_mode;
};
```

**Polygon Types:**

| Type | Value | Description | `.permutation` Use |
|------|-------|-------------|-------------------|
| `_polygon_is_normal` | 0 | Standard room | Unused |
| `_polygon_is_item_impassable` | 1 | Items cannot enter | Unused |
| `_polygon_is_monster_impassable` | 2 | Monsters cannot enter | Unused |
| `_polygon_is_hill` | 3 | King of the Hill zone | Unused |
| `_polygon_is_base` | 4 | CTF/team base | Team number |
| `_polygon_is_platform` | 5 | Elevator/door | Platform index |
| `_polygon_is_light_on_trigger` | 6 | Activates light | Lightsource index |
| `_polygon_is_platform_on_trigger` | 7 | Activates platform | Polygon index |
| `_polygon_is_light_off_trigger` | 8 | Deactivates light | Lightsource index |
| `_polygon_is_platform_off_trigger` | 9 | Deactivates platform | Polygon index |
| `_polygon_is_teleporter` | 10 | Transport player | Destination polygon |
| `_polygon_is_zone_border` | 11 | AI zone boundary | Unused |
| `_polygon_is_goal` | 12 | Level exit | Unused |
| `_polygon_is_visible_monster_trigger` | 13 | Activates monsters (sight) | Unused |
| `_polygon_is_invisible_monster_trigger` | 14 | Activates monsters (entry) | Unused |
| `_polygon_is_dual_monster_trigger` | 15 | Both trigger types | Unused |
| `_polygon_is_item_trigger` | 16 | Activates items in zone | Unused |
| `_polygon_is_automatic_exit` | 18 | Auto-exit on success | Unused |

> **Source:** `map.h:571` for `struct polygon_data`, `map.h:534-555` for polygon types

### Complete Example: A 4-Polygon Level

```
Real-world example showing all data structures together:

         e0 ──────L0────── e1
         │                  │
         │                  │
        L3    Polygon 0    L1
         │   (room center)  │
         │                  │
         │                  │
         e3 ──────L2────── e2
              ╱        ╲
             ╱          ╲
           L4            L5
           ╱              ╲
         e4 ───── L6 ───── e5

Data in memory:

Endpoints array (6 total):
  [0]: {x=512, y=512, ...}     // Top-left
  [1]: {x=1024, y=512, ...}    // Top-right
  [2]: {x=1024, y=1024, ...}   // Right
  [3]: {x=512, y=1024, ...}    // Bottom-left of main room
  [4]: {x=256, y=1280, ...}    // Far left
  [5]: {x=768, y=1280, ...}    // Far right

Lines array (7 total):
  [0]: {endpoints={0,1}, cw_owner=0, ccw_owner=-1}  // Top (solid)
  [1]: {endpoints={1,2}, cw_owner=0, ccw_owner=-1}  // Right (solid)
  [2]: {endpoints={2,3}, cw_owner=0, ccw_owner=1}   // Portal to poly 1
  [3]: {endpoints={3,0}, cw_owner=0, ccw_owner=-1}  // Left (solid)
  [4]: {endpoints={3,4}, cw_owner=1, ccw_owner=-1}  // Extension left
  [5]: {endpoints={2,5}, cw_owner=1, ccw_owner=-1}  // Extension right
  [6]: {endpoints={4,5}, cw_owner=1, ccw_owner=-1}  // Far edge (solid)

Polygons array:
  [0]: {vertex_count=4, endpoints={0,1,2,3}, lines={0,1,2,3},
        adjacent={-1,-1,1,-1}, floor=0, ceiling=1024}  // Main room
  [1]: {vertex_count=4, endpoints={3,2,5,4}, lines={2,5,6,4},
        adjacent={0,-1,-1,-1}, floor=0, ceiling=1024}  // Southern room

Traversal example - player walks from Polygon 0 to Polygon 1:
  1. Player in Polygon 0 crosses Line 2
  2. Line 2 has: clockwise_owner=0, counterclockwise_owner=1
  3. Since we came from polygon 0 (CW), we enter polygon 1 (CCW)
  4. Now player is in Polygon 1

This connectivity allows O(1) traversal - no searching needed!
```

---

## 4.5 Dynamic Elements

Marathon's world isn't static—doors open, platforms rise, water levels change.

### Platforms (Moving Geometry)

Platforms handle doors, elevators, and crushers:

```c
// From platforms.h:201 - 128 bytes per platform
struct platform_data {
    short type;          // Door, elevator, etc.
    word static_flags;   // Initial configuration
    short speed;         // Movement speed
    short delay;         // Pause at endpoints

    // Height ranges
    world_distance minimum_floor_height;
    world_distance maximum_floor_height;
    world_distance minimum_ceiling_height;
    world_distance maximum_ceiling_height;

    // Current state
    word dynamic_flags;  // Active, extending, contracting
    world_distance floor_height;
    world_distance ceiling_height;

    short polygon_index; // Which polygon this platform controls
    short tag;           // For scripted triggers

    // Sound and activation
    short parent_platform_index;
};
```

**Platform Types:**
- `_platform_is_spht_door` - Standard door (opens quickly)
- `_platform_is_heavy_spht_door` - Heavy door (opens slowly)
- `_platform_is_spht_platform` - Elevator
- `_platform_is_pfhor_door` - Alien door style
- `_platform_is_pfhor_platform` - Alien elevator

**Movement Speeds:**
```c
#define _very_slow_platform  (WORLD_ONE/(4*TICKS_PER_SECOND))  // ~8.5 units/sec
#define _slow_platform       (WORLD_ONE/(2*TICKS_PER_SECOND))  // ~17 units/sec
#define _fast_platform       (2*_slow_platform)                // ~34 units/sec
#define _very_fast_platform  (2*_fast_platform)                // ~68 units/sec
```

> **Source:** `platforms.h:201` for `struct platform_data`

### Media (Liquids)

Water, lava, and other liquids:

```c
// From media.h:61 - 32 bytes per media
struct media_data {
    short type;           // Water, lava, goo, sewage, jjaro
    word flags;

    // Height animation
    short light_index;    // Controls height animation!

    // Flow properties
    angle current_direction;
    world_distance current_magnitude;

    // Height bounds
    world_distance low, high;

    // Position
    world_point2d origin;
    world_distance height;  // Current level

    // Visual
    world_distance minimum_light_intensity;
    shape_descriptor texture;
    short transfer_mode;
};
```

**Clever Height Animation:**

Marathon ties media height to light intensity:
```c
height = low + FIXED_INTEGRAL_PART((high - low) * get_light_intensity(light_index))
```

This allows water to rise and fall by simply animating a light source! A strobe light makes choppy waves. A smooth fade creates a rising tide.

**Media Types:**

| Type | Effect |
|------|--------|
| `_media_water` | Slows movement, can drown |
| `_media_lava` | Damages player |
| `_media_goo` | Sticky, damages |
| `_media_sewage` | Damages slowly |
| `_media_jjaro` | Alien liquid |

> **Source:** `media.h:61` for `struct media_data`

### Lighting

Dynamic lights affect both ambience and media height:

```c
// Light types
enum {
    _normal_light,   // Standard on/off
    _strobe_light,   // Blinking
    _media_light     // Controls liquid height
};

// Lighting functions (how intensity changes over time)
enum {
    _constant_lighting_function,  // Instant on/off
    _linear_lighting_function,    // Linear fade
    _smooth_lighting_function,    // Sine wave
    _flicker_lighting_function    // Random noise
};

// Light state machine
enum {
    _light_becoming_active,
    _light_primary_active,
    _light_secondary_active,
    _light_becoming_inactive,
    _light_primary_inactive,
    _light_secondary_inactive
};
```

Each light state has independent transition parameters (period, intensity, function), allowing complex lighting effects.

---

## 4.6 Object Placement and Spawning

Marathon uses a sophisticated system for placing and respawning objects.

### Map Objects (Initial Placement)

```c
#define MAXIMUM_SAVED_OBJECTS 384

enum {  // Object types in map files
    _saved_monster,      // .index is monster type
    _saved_object,       // .index is scenery type
    _saved_item,         // .index is item type
    _saved_player,       // .index is team bitfield
    _saved_goal,         // .index is goal number
    _saved_sound_source  // .index is source type, .facing is volume
};

struct map_object {
    short type;
    short index;           // Subtype (which monster/item)
    short facing;          // Initial angle
    short polygon_index;   // Which room
    world_point3d location; // Position (.z is delta from polygon floor)
    word flags;
};
```

**Object Flags:**
- `_map_object_is_invisible` (0x0001) - Starts hidden
- `_map_object_hanging_from_ceiling` (0x0002) - Suspended (for position calculation)
- `_map_object_is_blind` (0x0004) - Monster won't react to sight
- `_map_object_is_deaf` (0x0008) - Monster won't react to sound
- `_map_object_floats` (0x0010) - Hovering object
- `_map_object_is_network_only` (0x0020) - Multiplayer only

### Dynamic Respawning

```c
struct object_frequency_definition {
    word flags;
    short initial_count;   // At map start
    short minimum_count;   // Maintained minimum
    short maximum_count;   // Cap
    short random_count;    // Max random spawns
    word random_chance;    // Probability [0-65535]
};
```

**Placement Flags:**
- `_reappears_in_random_location` - Objects respawn at random positions when destroyed

**Respawn Algorithm** (from `recreate_objects()`):
1. Check current object count vs minimum_count
2. If below minimum and `_monsters_replenish` game option enabled:
   - Select random polygon weighted by polygon area
   - Find random point within selected polygon
   - Verify visibility requirements (some items must be visible to player)
   - Verify object type is valid for that polygon
   - Spawn new object instance

### Difficulty Modulation

Marathon adjusts monster spawning based on difficulty level:

- **Wuss/Easy difficulty**: Randomly drop 1/4 to 1/8 of monsters at map start
- **Major monsters**: Demoted to minor variants on lower difficulties
- Affects spawn density, not individual monster abilities

This system ensures:
- Maps never become completely empty of monsters/items
- Difficulty scales appropriately
- Players always have resources available
- Multiplayer games maintain balance

---

## 4.7 Summary

Marathon represents its world as a graph of connected polygons, enabling:

**Efficient Traversal:**
- O(1) adjacent polygon lookup
- No searching or tree traversal needed
- Perfect for both rendering (portal culling) and physics

**Flexible Level Design:**
- Arbitrary polygon shapes (up to 8 vertices)
- Variable floor/ceiling heights
- Dynamic elements (platforms, liquids)

**Rich Environment:**
- Multiple texture layers per wall
- Dynamic lighting with state machines
- Media (liquids) with flow and height animation

### Key Data Structure Sizes

| Structure | Size | Max Count | Purpose |
|-----------|------|-----------|---------|
| `world_point2d` | 4 bytes | N/A | Base 2D coordinate |
| `endpoint_data` | 16 bytes | 2048 | Vertices |
| `line_data` | 32 bytes | 4096 | Edges |
| `side_data` | 64 bytes | 8192 | Wall textures |
| `polygon_data` | 128 bytes | 1024 | Rooms |
| `platform_data` | 128 bytes | 64 | Moving geometry |
| `media_data` | 32 bytes | 16 | Liquids |

### Key Source Files

| File | Purpose |
|------|---------|
| `map.h` | All core structure definitions |
| `map.c` | Level loading and queries |
| `map_constructors.c` | Building map data |
| `platforms.h/c` | Dynamic platforms |
| `media.h/c` | Liquid handling |
| `lightsource.h/c` | Dynamic lighting |

### Source Reference Summary

| Structure | Location |
|-----------|----------|
| `struct endpoint_data` | map.h:378 |
| `struct line_data` | map.h:416 |
| `struct side_data` | map.h:499 |
| `struct polygon_data` | map.h:571 |
| `struct platform_data` | platforms.h:201 |
| `struct media_data` | media.h:61 |

---

*Next: [Chapter 5: The Rendering System](05_rendering.md) - How Marathon turns this world into pixels*
