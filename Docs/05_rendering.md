# Chapter 5: The Rendering System

## Seeing the Marathon World

> **For Porting:** The core renderer (`render.c`, 3,879 lines) is 99% portable C! Changes needed:
> - Replace `world_pixels` GWorld with your framebuffer (32-bit ARGB)
> - Modify `scottish_textures.c` to write 32-bit pixels instead of 8-bit palette indices
> - Remove/stub assembly texture mappers—C fallbacks exist
> - See `screen.c` for framebuffer setup requiring platform replacement
>
> **Terminology note:** This chapter uses `world_pixels`, `screen`, `destination`, and `framebuffer` interchangeably for the render target buffer. See **[Appendix A: Glossary → Graphics Buffer Terminology](appendix_a_glossary.md#graphics-buffer-terminology)** for detailed clarification of these terms.

---

## 5.1 What Problem Are We Solving?

Marathon needs to turn a 3D world into a 2D image, 30 times per second, on hardware from 1995. This is the fundamental challenge of any 3D game engine.

**The constraints:**
- No GPU (pure software rendering)
- ~25 MHz 68040 processor
- ~640×480 resolution at 8-bit color
- Complex indoor environments with multiple rooms
- Must maintain 15-30 FPS

**What makes this hard?**

A naive approach would be to draw every polygon in the level. But Marathon levels can have 500-1000 polygons. At 30 FPS, that's potentially 30,000 polygon renders per second—far too slow for 1995 hardware.

The key insight: **you can only see a small portion of the level at any moment**. Walls block your view. If we can figure out what's *actually* visible, we only need to draw 50-100 polygons per frame.

This is the **visibility problem**, and Marathon solves it elegantly with **portal-based rendering**.

### How Marathon Differs from Wolfenstein-Style Raycasting

Marathon famously advertised "non-orthogonal walls"—arbitrary angles instead of the 90° grid that Wolfenstein 3D and Pathways Into Darkness required. This wasn't just a data format change; it required a fundamentally different rendering approach.

**Wolfenstein 3D (1992): DDA Raycasting**

Wolfenstein casts one ray per screen column, using the DDA (Digital Differential Analyzer) algorithm to step through a uniform grid until hitting a wall:

```
Wolfenstein's Grid-Based Raycasting:

    ┌───┬───┬───┬───┬───┬───┐
    │   │   │ █ │ █ │   │   │
    ├───┼───┼───┼───┼───┼───┤
    │   │   │ █ │ █ │   │   │     Ray cast from player (@)
    ├───┼───┼───┼─·─┼─··│   │     steps through grid cells
    │   │   │ @ →→→→→→·→│···│     until it hits a wall (█)
    ├───┼───┼───┼───┼───┼───┤
    │   │   │   │   │   │   │     Only 90° walls possible!
    └───┴───┴───┴───┴───┴───┘
```

- Cast 320 rays (one per column) per frame
- DDA efficiently steps through grid intersections
- Find exact ray-wall intersection point
- **Limitation**: Walls must align to grid (orthogonal only)

**Marathon (1994): Portal-Based Projection**

Marathon doesn't cast rays to find wall intersections. Instead, it projects wall endpoints to screen space and clips them against portal boundaries:

```
Marathon's Portal-Based Approach (top-down view):

    ┌──────────────────╥──────────────────┐
    │                  ║                  │
    │    Polygon A     ║     Polygon B    │
    │                  ║                  │
    │    @ ────────────╫──────────►       │  @ = Player, looking right
    │    (player)      ║                  │  ║ = Portal (shared edge)
    │                  ║                  │
    │                  ║                  │
    │                  ║                  │
    └──────────────────╨──────────────────┘

    Walls can be ANY angle! No grid constraints.

1. Start in player's polygon (A)
2. For each edge: project endpoints to screen X
3. If edge is portal (║) AND visible: recurse into neighbor (B)
4. If edge is solid: texture map the wall span
```

**Key Differences:**

| Aspect | Wolfenstein (DDA) | Marathon (Portal) |
|--------|-------------------|-------------------|
| **Per-frame work** | Cast ray per screen column | Project polygon endpoints |
| **Wall intersection** | Calculate ray-wall hit point | No intersection—clip endpoints |
| **Wall angles** | 90° only (grid-aligned) | Arbitrary angles |
| **Data structure** | 2D grid of cells | Polygon connectivity graph |
| **Complexity** | O(columns × grid_steps) | O(visible_polygons × edges) |

**Why This Matters:**

The cross product test Marathon uses (see Section 5.4) determines which polygon edge a ray *exits through*—not where it intersects. This is for building the visibility tree, not for rendering. Actual wall rendering uses endpoint projection and span-based texture mapping, more like a software rasterizer than a raycaster.

```c
// Wolfenstein: find WHERE ray hits wall
intersection = ray_origin + t * ray_direction;  // solve for t

// Marathon: project wall endpoints, clip to screen bounds
screen_x1 = half_screen + (endpoint1.y * scale) / endpoint1.x;
screen_x2 = half_screen + (endpoint2.y * scale) / endpoint2.x;
// Then texture map columns from screen_x1 to screen_x2
```

**Wall Projection and Clipping Visualized:**

```
WORLD SPACE (top-down view)                    SCREEN SPACE (what you see)

  ═══════════════════════════════════           ┌─────────────────────┐
  ║  e2──────────e3                 ║           │                     │
  ║     Wall B (far)                ║           │   ┃ Wall A  ┃       │
   ╲                               ╱            │  B┃█████████┃       │
    ╲                             ╱             │ ┃█┃█████████┃       │
     ╲    e0━━━━━━━━━━━e1        ╱              │ ┃█┃█████████┃       │
      ╲      Wall A (near)      ╱               │ ┃█┃█████████┃       │
       ╲                       ╱                │   ┃█████████┃       │
        ╲                     ╱                 │   e0       e1       │
         ╲                   ╱                  └─────────────────────┘
          ╲   View Cone     ╱
           ╲               ╱                  Wall B (e2) visible on left,
            ╲             ╱                   partially occluded by Wall A.
             ╲           ╱                    Wall A fully visible in center.
              ╲         ╱
               ╲       ╱
                ╲     ╱
                 ╲   ╱
                  ╲ ╱
                   @
                Player
                (Apex)

Step 1: Transform endpoints to camera space (rotate by -player_angle)
Step 2: Project each endpoint:  screen_x = center + (local_y × scale) / local_x
Step 3: Wall A (near): fully visible, projects to center of screen
        Wall B (far): e2 visible on left, e3 occluded behind Wall A

Painter's algorithm: Wall B drawn first, then Wall A overwrites the overlap.


PORTAL CLIPPING:

    World: Room A → Portal → Room B             Screen with portal clip window

    ┌──────────┬───Portal───┬──────────┐    ┌─────────────────────────────┐
    │          │ p0     p1  │          │    │      Portal bounds          │
    │  Room A  │     ↓      │  Room B  │    │      ┃ p0    p1 ┃           │
    │          │            │          │    │      ┃    ↓     ┃           │
    │    @─────┼────────────┼─────     │    │ Room A  ┃ Room B ┃           │
    │  Player  │            │          │    │ visible ┃visible ┃           │
    └──────────┴────────────┴──────────┘    │      ┃█████████┃           │
                                            │      ┃█ wall  █┃           │
    Wall in Room B spans w0────────w1       │      ┃█████████┃           │
                                            │   w0'┃ clipped ┃w1         │
                                            └─────────────────────────────┘

    Wall endpoints w0, w1 project to screen, but get CLIPPED to portal bounds.
    Only the portion between p0 and p1 is rendered—the rest is occluded by Room A's walls.
```

> **Implementation Note:** The diagram above is simplified for teaching. Marathon's actual clipping is more sophisticated:
>
> 1. Each `clipping_window_data` stores both **world-space clip vectors** (`left`, `right`, `top`, `bottom`) and **screen-space bounds** (`x0`, `x1`, `y0`, `y1`)
> 2. Geometric clipping happens in **world space** before projection, using functions like `xy_clip_horizontal_polygon()` (render.c:2372-2373)
> 3. Clipped vertices receive flags (`_clip_left`, `_clip_right`, etc.)
> 4. During screen projection, clipped vertices snap directly to screen bounds:
>    ```c
>    // render.c:2393-2394
>    case _clip_left:  screen->x = window->x0; break;
>    case _clip_right: screen->x = window->x1; break;
>    ```
>
> This world-space clipping correctly handles perspective—a vertex clipped to a portal edge in 3D space maps exactly to the portal's screen edge.

This architectural difference enabled Marathon's complex environments with overlapping spaces, sloped surfaces viewed through windows, and the iconic "5D space" tricks that mappers exploited.

---

## 5.2 Understanding Portal Rendering

### The Core Idea

Imagine you're standing in a room. You can see:
1. The walls, floor, and ceiling of your current room
2. Through any doorways, you can see into adjacent rooms
3. Through *those* rooms' doorways, you might see even further

Each doorway acts as a **portal**—a window into another space. We only need to draw what's visible through these portals.

### A Simple Mental Model

```
You're standing in Room A, looking through a doorway into Room B:

        ┌─────────────────────────────────────────┐
        │                                         │
        │              ROOM A                     │
        │                                         │
        │                 @  ← You                │
        │                 │                       │
        │                 │ (your view)           │
        │                 ▼                       │
        │         ╔═══════════════╗               │
        │         ║    PORTAL     ║ ← Doorway     │
        │         ╚═══════════════╝               │
        │                 │                       │
        └─────────────────┼───────────────────────┘
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │                                          │
        │               ROOM B                     │
        │                                          │
        │    You can only see the part of Room B   │
        │    that's visible THROUGH the portal!    │
        │                                          │
        └──────────────────────────────────────────┘
```

The portal *clips* your view of Room B. You can't see the parts of Room B that are "around the corner" from the doorway.

### Why This Is Fast

Instead of testing every polygon against the view frustum, we:
1. Start with the room we're standing in (always fully visible)
2. For each portal (doorway) in that room, check if it's in our field of view
3. If visible, recursively render what's through that portal
4. Each portal *narrows* the visible region, so deeper rooms render less

This naturally limits rendering to only what's visible.

---

## 5.3 Let's Build: A Minimal Portal Renderer

Before diving into Marathon's implementation, let's build a simplified portal renderer to understand the concepts. We'll use pseudocode first, then modern C.

### Step 1: Data Structures

A portal-based world needs:
- **Sectors** (rooms): polygonal regions with floor/ceiling heights
- **Walls**: the edges of sectors, some solid, some portals
- **Portals**: walls that connect to other sectors

```
PSEUDOCODE: Basic data structures

Sector:
    walls[]          -- list of walls forming this sector
    floor_height     -- Z coordinate of floor
    ceiling_height   -- Z coordinate of ceiling

Wall:
    start_point      -- (x, y) of wall start
    end_point        -- (x, y) of wall end
    is_portal        -- true if this is a doorway
    portal_to        -- if portal, which sector it leads to
    texture          -- if solid wall, which texture to draw
```

Now in modern C:

```c
// Modern C implementation
typedef struct {
    float x, y;
} Vec2;

typedef struct {
    Vec2 start, end;
    bool is_portal;
    int portal_to_sector;    // -1 if solid wall
    int texture_id;
} Wall;

typedef struct {
    Wall *walls;
    int wall_count;
    float floor_z, ceiling_z;
} Sector;

typedef struct {
    Sector *sectors;
    int sector_count;
} World;
```

**Marathon's approach:** Marathon calls sectors "polygons" and stores them differently. Instead of each sector containing its walls, Marathon uses a shared pool of lines and endpoints, with polygons referencing them by index. This saves memory (lines shared between adjacent polygons) but adds complexity.

> **Source:** `map.h:571` for `struct polygon_data`, `map.h:416` for `struct line_data`

### Step 2: The Render Loop

The core algorithm is surprisingly simple:

```
PSEUDOCODE: Portal rendering

function render_world(player_position, player_angle):
    current_sector = find_sector_containing(player_position)

    -- Start with full screen as our "window"
    render_sector(current_sector,
                  screen_left=0,
                  screen_right=SCREEN_WIDTH)

function render_sector(sector, screen_left, screen_right):
    -- Don't render if window is empty
    if screen_left >= screen_right:
        return

    for each wall in sector.walls:
        -- Transform wall endpoints to screen space
        wall_screen_left, wall_screen_right = project_wall(wall)

        -- Clip to our current window
        clipped_left = max(wall_screen_left, screen_left)
        clipped_right = min(wall_screen_right, screen_right)

        if clipped_left >= clipped_right:
            continue  -- Wall not visible in our window

        if wall.is_portal:
            -- Recursively render what's through the portal
            -- The portal becomes the NEW, NARROWER window
            render_sector(wall.portal_to_sector,
                         clipped_left,
                         clipped_right)
        else:
            -- Draw the solid wall
            draw_wall(wall, clipped_left, clipped_right)

    -- Draw floor and ceiling for this sector
    draw_floor(sector, screen_left, screen_right)
    draw_ceiling(sector, screen_left, screen_right)
```

Now in modern C:

```c
// Screen clipping window
typedef struct {
    int x_left, x_right;  // Horizontal bounds on screen
} ClipWindow;

void render_sector(World *world, int sector_id, ClipWindow window,
                   Vec2 camera_pos, float camera_angle) {
    if (window.x_left >= window.x_right) return;

    Sector *sector = &world->sectors[sector_id];

    for (int i = 0; i < sector->wall_count; i++) {
        Wall *wall = &sector->walls[i];

        // Project wall to screen coordinates
        int wall_left, wall_right;
        if (!project_wall(wall, camera_pos, camera_angle,
                          &wall_left, &wall_right)) {
            continue;  // Behind camera
        }

        // Clip to current window
        int clipped_left = (wall_left > window.x_left) ? wall_left : window.x_left;
        int clipped_right = (wall_right < window.x_right) ? wall_right : window.x_right;

        if (clipped_left >= clipped_right) continue;

        if (wall->is_portal) {
            // Recurse through portal with narrower window
            ClipWindow sub_window = {clipped_left, clipped_right};
            render_sector(world, wall->portal_to_sector, sub_window,
                         camera_pos, camera_angle);
        } else {
            // Draw solid wall
            draw_wall_slice(wall, clipped_left, clipped_right);
        }
    }

    draw_floor_ceiling(sector, window);
}
```

**Marathon's approach:** Marathon uses a more sophisticated version with:
- A tree structure (`node_data`) instead of direct recursion
- Ray casting to build the visibility tree (NOT for rendering—rays determine which polygon edges to cross when tracing through portals, see Section 5.4)
- Accumulated clipping windows that handle vertical clipping too

> **Source:** `render.c:702` for `build_render_tree()`, `render.c:222-236` for `struct node_data`, `render.c:207-213` for `struct clipping_window_data`

### Step 3: Projecting Walls to Screen Space

To know where a wall appears on screen, we need to transform its 3D coordinates to 2D screen coordinates.

```
PSEUDOCODE: Wall projection

function project_wall(wall, camera_pos, camera_angle):
    -- Transform endpoints relative to camera
    local_start = rotate_point(wall.start - camera_pos, -camera_angle)
    local_end = rotate_point(wall.end - camera_pos, -camera_angle)

    -- If both points are behind camera, wall is invisible
    if local_start.y <= 0 AND local_end.y <= 0:
        return NOT_VISIBLE

    -- Clip against near plane if needed
    if local_start.y <= 0:
        local_start = clip_to_near_plane(local_start, local_end)
    if local_end.y <= 0:
        local_end = clip_to_near_plane(local_end, local_start)

    -- Perspective divide: x_screen = x_world / y_world
    screen_left = SCREEN_CENTER_X + (local_start.x * FOV_SCALE) / local_start.y
    screen_right = SCREEN_CENTER_X + (local_end.x * FOV_SCALE) / local_end.y

    -- Ensure left < right
    if screen_left > screen_right:
        swap(screen_left, screen_right)

    return (screen_left, screen_right)
```

Modern C implementation:

```c
// Rotate a point around origin by angle (in radians)
Vec2 rotate_point(Vec2 p, float angle) {
    float cos_a = cosf(angle);
    float sin_a = sinf(angle);
    return (Vec2){
        p.x * cos_a - p.y * sin_a,
        p.x * sin_a + p.y * cos_a
    };
}

// Project wall to screen X coordinates
// Returns false if wall is behind camera
bool project_wall(Wall *wall, Vec2 camera_pos, float camera_angle,
                  int *out_left, int *out_right) {

    // Transform to camera-relative coordinates
    Vec2 rel_start = {wall->start.x - camera_pos.x, wall->start.y - camera_pos.y};
    Vec2 rel_end = {wall->end.x - camera_pos.x, wall->end.y - camera_pos.y};

    Vec2 local_start = rotate_point(rel_start, -camera_angle);
    Vec2 local_end = rotate_point(rel_end, -camera_angle);

    // In our coordinate system, +Y is forward
    // Both behind camera?
    if (local_start.y <= 0.1f && local_end.y <= 0.1f) {
        return false;
    }

    // Clip to near plane (simplified - full impl would interpolate)
    const float NEAR = 0.1f;
    if (local_start.y < NEAR) local_start.y = NEAR;
    if (local_end.y < NEAR) local_end.y = NEAR;

    // Perspective projection
    const float FOV_SCALE = SCREEN_WIDTH * 0.5f;  // Adjust for FOV
    const float CENTER_X = SCREEN_WIDTH / 2.0f;

    float screen_x1 = CENTER_X + (local_start.x * FOV_SCALE) / local_start.y;
    float screen_x2 = CENTER_X + (local_end.x * FOV_SCALE) / local_end.y;

    // Ensure proper ordering
    if (screen_x1 > screen_x2) {
        float tmp = screen_x1; screen_x1 = screen_x2; screen_x2 = tmp;
    }

    *out_left = (int)screen_x1;
    *out_right = (int)screen_x2;
    return true;
}
```

**Marathon's approach:** Marathon uses fixed-point math (16.16 format) instead of floating-point for deterministic behavior across different machines. The projection happens in `calculate_endpoint_clipping_information()` in render.c. Marathon also uses lookup tables for trigonometry (sine/cosine tables) instead of `sinf()`/`cosf()` calls.

> **Source:** `render.c:1623` for `calculate_endpoint_clipping_information()`, `render.c:745` for screen projection, `world.c:112-127` for `transform_point2d()`, `world.c:36-37` for trig tables

```c
// Marathon's fixed-point projection (render.c:745)
// Uses integer math throughout
screen_x = view->half_width +
           (transformed_y * view->world_to_screen_x) / transformed_x;
```

---

## 5.4 Marathon's Visibility System

Now that we understand the basic concept, let's see how Marathon actually implements portal rendering. Marathon's system is more sophisticated because it needs to:

1. Handle arbitrary polygon shapes (not just rectangular rooms)
2. Support vertical portals (windows, platforms at different heights)
3. Build a rendering order for proper depth sorting
4. Maximize performance on slow hardware

### The Render Tree

Instead of direct recursion, Marathon builds a **tree structure** representing what's visible:

```
                     Node 0
                (Player's Room)
                [full screen width]
                       │
         ┌─────────────┼─────────────┐
         │             │             │
      Node 1        Node 2        Node 3
   (Left Door)   (Center Door)  (Right Door)
   [x: 0-200]    [x: 200-450]   [x: 450-640]
         │             │
     ┌───┴───┐     ┌───┴───┐
  Node 1.1  1.2   Node 2.1  2.2
  [0-100] [100-200]  ...    ...
```

Each node stores:
- Which polygon (sector) to render
- The clipping window (visible region on screen)
- Links to child nodes (what's visible through this node's portals)

**Why a tree instead of recursion?**
1. Can be traversed multiple ways (build order vs render order)
2. Easier to sort for back-to-front rendering
3. Can accumulate clipping information across multiple paths to the same polygon

### Key Data Structures

```c
// From render.c:222-236 - the node structure (simplified)
struct node_data {
    word flags;
    short polygon_index;              // Which polygon this node represents

    // Clipping information accumulated from portal traversal
    short clipping_endpoint_count;
    short clipping_endpoints[MAXIMUM_CLIPPING_ENDPOINTS_PER_NODE];
    short clipping_line_count;
    short clipping_lines[MAXIMUM_CLIPPING_LINES_PER_NODE];

    // Tree links
    struct node_data *parent;
    struct node_data *children;
    struct node_data *siblings;
};
```

### The Build Algorithm

Marathon's `build_render_tree()` works as follows:

```
1. Create root node for player's polygon (always fully visible)

2. Cast rays at the left and right edges of the screen
   - These define the initial view frustum
   - Use bias flags to handle rays exactly hitting vertices

3. Process polygons in a breadth-first manner:
   For each polygon in the queue:
     a. Cast rays through each vertex of the polygon
     b. When a ray crosses a portal (non-solid line):
        - Determine which adjacent polygon the ray enters
        - Create a child node for that polygon
        - Narrow the clipping window based on the portal edges
        - Add the new polygon to the queue

4. Continue until no more visible polygons are found
```

**Portal Visibility Test**:

```
                   ═══════════════════════
                   ║       Screen        ║
                    ╲                   ╱
                     ╲                 ╱
                      ╲   View Cone   ╱
                       ╲             ╱
                        ╲           ╱
          Left Ray ──────╲         ╱────── Right Ray
                          ╲       ╱
                           ╲     ╱
                            ╲   ╱
                             ╲ ╱
                              @
                           (Player)
                            Apex

1. Cast ray at left screen edge (counterclockwise bias*)
2. Cast ray at right screen edge (clockwise bias*)
3. Any polygon touched by rays in this cone is potentially visible
4. Build clipping window from portal edges

*"Bias" determines which side of a line to cross when a ray exactly
 hits a vertex/endpoint. From render.c:695-700:
   _clockwise_bias: cross the line clockwise from this endpoint
   _counterclockwise_bias: cross the line counterclockwise from endpoint
 This prevents ambiguity when rays pass exactly through vertices.
```

### Ray Casting Through the Polygon Graph

The key operation is `cast_render_ray()`, which traces a ray through the world:

```
         Polygon A              Polygon B              Polygon C
    ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
    │              │       │              │       │              │
    │   Player     │       │              │       │   Target     │
    │     @────────┼───────┼──────────────┼───────┼────>*        │
    │              │       │              │       │              │
    └──────────────┘       └──────────────┘       └──────────────┘
         Portal 1               Portal 2
```

To determine which edge a ray exits through, Marathon uses the **cross product test**:

```c
// Test if ray passes through edge (e0 → e1)
// See render.c:864-1000 for next_polygon_along_line()
long cross = (ray.x - e0.x) * (e1.y - e0.y) -
             (ray.y - e0.y) * (e1.x - e0.x);

if (cross > 0) {
    // Ray exits through this edge
    // If edge is a portal, continue into adjacent polygon
    // If edge is solid, ray terminates
}
```

**Visual intuition for the cross product test:**

```
Which side of line e0→e1 is point P on?

                          e1
                           *
                          ╱
                         ╱
                        ╱
          LEFT side    ╱    RIGHT side
         (cross < 0)  ╱    (cross > 0)
                     ╱
                    ╱           * P
                   ╱
                  ╱
                 *
                e0

cross = (P.x - e0.x) × (e1.y - e0.y) - (P.y - e0.y) × (e1.x - e0.x)

If cross > 0: P is on the RIGHT of line (looking from e0 toward e1)
If cross < 0: P is on the LEFT of line
If cross = 0: P is exactly ON the line
```

The cross product gives us the signed area of the triangle formed by the edge and the ray point. Positive means the point is on the right side of the edge (when walking from e0 to e1), which in Marathon's counter-clockwise polygon winding means "outside" the polygon.

### Source Reference

The visibility system lives in `render.c`:

| Function | Line | Purpose |
|----------|------|---------|
| `render_view()` | 497 | Main render entry point |
| `build_render_tree()` | 702 | Builds visibility tree via ray casting |
| `cast_render_ray()` | 775 | Traces ray through polygon graph |
| `next_polygon_along_line()` | 864 | Cross product test for edge crossing |
| `calculate_endpoint_clipping_information()` | 1623 | Projects vertices to screen |
| `sort_render_tree()` | 1116 | Converts tree to back-to-front array |

---

## 5.5 Understanding 3D Projection

Before we can render anything, we need to understand how 3D points become 2D screen coordinates. This section explains the math without assuming prior knowledge.

### The Camera Model

Think of the screen as a window you're looking through:

```
Side view (X-Z plane, Y pointing into page):

              +Z (up)
               ↑
               │    Screen
               │      │
    Eye ───────┼──────┼────────────────► +X (forward/depth)
               │      │
               │
               ↓ -Z (down)

Top view (X-Y plane, Z pointing up out of page):

              +Y (right)
               ↑
               │    ┌─────┐
               │    │     │
    Eye ───────┼────┤     ├──────► +X (forward/depth)
               │    │     │
               │    └─────┘
               ↓     Screen
              -Y (left)
```

Points in the world project onto the screen based on their angle from the eye.

### The Perspective Divide

The fundamental equation of perspective projection:

```
screen_x = world_x / world_depth
screen_y = world_y / world_depth
```

Things farther away (`world_depth` is larger) appear smaller on screen. This division by depth is called the **perspective divide**.

**Visual intuition:**

```
Two objects at different depths, same world size:

        ┌───┐                    Near object
        │   │                    depth = 2
        │   │                    screen_size = size / 2 = 0.5
        └───┘

        ┌─┐                      Far object
        │ │                      depth = 4
        └─┘                      screen_size = size / 4 = 0.25
```

### Marathon's Projection Formula

Marathon transforms points in two steps:

**Step 1: Transform to View Space**

Convert world coordinates to camera-relative coordinates:

```c
// Translate so camera is at origin
relative_x = world_x - camera_x;
relative_y = world_y - camera_y;
relative_z = world_z - camera_z;

// Rotate so camera faces +X direction
// (Marathon uses yaw angle for horizontal rotation)
view_x = relative_x * cos(yaw) + relative_y * sin(yaw);  // depth
view_y = -relative_x * sin(yaw) + relative_y * cos(yaw); // left/right
view_z = relative_z;                                       // up/down
```

After this transform:
- `view_x` is depth (positive = in front of camera)
- `view_y` is left/right (positive = right of camera)
- `view_z` is up/down (positive = above camera)

**Step 2: Project to Screen**

```c
// Only project if in front of camera
if (view_x <= 0) {
    // Behind camera - not visible
    return;
}

// Perspective divide + scale to screen pixels
screen_x = screen_center_x + (view_y * fov_scale) / view_x;
screen_y = screen_center_y - (view_z * fov_scale) / view_x;
//                         ^ Minus because screen Y goes down
```

The `fov_scale` factor controls field of view—larger values = wider view.

### Pitch Calculation (Looking Up/Down)

Marathon adds vertical look via `dtanpitch`. Here's how it works:

```
Looking level (pitch = 0):
                                Screen
    World Z                  ┌─────────┐
       ↑                     │         │
       │                     │    ●────┼── half_height (screen center)
   ────┼────  Z = eye_level  │         │
       │                     │         │
       │                     └─────────┘
       └─→ View direction    dtanpitch = 0

Looking up (pitch > 0):
                                Screen
    World Z                  ┌─────────┐
       ↑  ╱                  │         │
       │ ╱ pitch angle       │         │
   ────┼────  Z = eye_level  │    ●────┼── center shifted DOWN
       │                     │         │
       └─→ View direction    └─────────┘
                             dtanpitch > 0 (shows more ceiling)

Looking down (pitch < 0):
                                Screen
    World Z                  ┌─────────┐
       │                     │    ●────┼── center shifted UP
   ────┼────  Z = eye_level  │         │
       │ ╲                   │         │
       ↓  ╲ pitch angle      └─────────┘
          └─→ View direction dtanpitch < 0 (shows more floor)
```

**dtanpitch Calculation:**

```c
// Pitch is stored as a fixed-point angle
// tan(pitch) shifts all vertical coordinates
dtanpitch = world_to_screen_y * tan(pitch_angle);
screen_y = screen_center_y - (view_z * fov_scale_y) / view_x + dtanpitch;

// Example with pitch = +15 degrees (looking up):
// - world_to_screen_y = 300 (example value)
// - tan(15°) ≈ 0.268
// - dtanpitch = 300 * 0.268 = 80 pixels
// Result: All points shifted down 80 pixels on screen
//         → Looking up shows more ceiling
```

**Complete Screen Projection Formula:**

```
World Point (x, y, z) → Screen Pixel (screen_x, screen_y)

Step 1: Transform to view space
    view_x = (world.x - camera.x) * cos(yaw) + (world.y - camera.y) * sin(yaw)
    view_y = -(world.x - camera.x) * sin(yaw) + (world.y - camera.y) * cos(yaw)
    view_z = world.z - camera.z

Step 2: Perspective divide
    screen_x_raw = view_y / view_x  (horizontal position)
    screen_z_raw = view_z / view_x  (vertical position)

Step 3: Scale and center
    screen_x = half_width + (screen_x_raw * world_to_screen_x)
    screen_y = half_height - (screen_z_raw * world_to_screen_y) + dtanpitch
                                                                   ↑
                                                    Pitch compensation

Clipping:
- If screen_x < 0 or screen_x >= screen_width: off-screen
- If screen_y < 0 or screen_y >= screen_height: off-screen
- If view_x <= 0: behind camera
```

### Fixed-Point Math

**Marathon's approach:** Instead of floating-point (which was slow in 1995 and non-deterministic across machines), Marathon uses **fixed-point arithmetic**:

```c
// Marathon uses 16.16 fixed-point: 16 bits integer, 16 bits fraction
#define FIXED_ONE 65536  // 1.0 in fixed-point

// Convert float to fixed: multiply by 65536
fixed_t float_to_fixed(float f) {
    return (fixed_t)(f * 65536.0f);
}

// Convert fixed to int: shift right 16 bits
int fixed_to_int(fixed_t f) {
    return f >> 16;
}

// Multiply two fixed numbers: multiply, then shift back
fixed_t fixed_mul(fixed_t a, fixed_t b) {
    return (a * b) >> 16;
}
```

> **For complete fixed-point details:** See **[Appendix D: Fixed-Point Math](appendix_d_fixedpoint.md)** for multiplication, division, overflow handling, and conversion functions.

Marathon's trigonometry uses lookup tables instead of `sin()`/`cos()`:

```c
// Pre-computed tables with 512 entries (one per angle unit)
// See world.c:36-37 for declarations, world.c:172-187 for initialization
short cosine_table[512];
short sine_table[512];

// Marathon angles: 0-511 instead of 0-2π (512 = full circle)
// Lookup is just array access, no math needed
short cos_value = cosine_table[angle & 511];
```

**Why `& 511` works:** 512 is a power of two (2⁹), so `& 511` is equivalent to `% 512` but much faster. This bitwise trick only works with powers of two:
- `511` in binary is `111111111` (nine 1s)
- ANDing with this mask keeps only the lower 9 bits
- Result is always in range [0, 511], wrapping automatically

> **Source:** `world.h:118` for table declarations, `world.c:172-187` for table initialization, `world.h:21-28` for angle constants (NUMBER_OF_ANGLES=512)

---

## 5.6 Texture Mapping: Painting the Walls

Once we know which surfaces are visible and where they appear on screen, we need to fill them with textures. Marathon uses two different approaches:

1. **Affine mapping** for floors and ceilings (fast but approximate)
2. **Perspective-correct mapping** for walls (slower but accurate)

### Why Two Approaches?

Texture mapping means figuring out: "For each screen pixel, which texel (texture pixel) should I sample?"

The naive approach (affine mapping) linearly interpolates texture coordinates across the surface. This works well when the surface is roughly parallel to the screen, but causes visible distortion on surfaces at steep angles.

```
The problem with affine mapping on walls:

TOP-DOWN VIEW:                          FRONT VIEW (what you see):

         far ─────┐                              FAR         NEAR
                  │                             (left)      (right)
                  │ wall                          │           │
                  │ surface                       ▼           ▼
         near ────┘                         ┌─────────────────────┐
              ↑                             │░░│░░░│░░░░│████████│
           player                           │░░│░░░│░░░░│████████│
                                            │░░│░░░│░░░░│████████│  ← Wall height
                                            │░░│░░░│░░░░│████████│    GROWS toward
                                            │░░│░░░│░░░░│████████│    near side!
                                            │░░│░░░│░░░░│████████│
                                            └─────────────────────┘
                                             ↑small        large↑

Two perspective effects on walls:
1. HORIZONTAL: Near parts span more screen pixels (texels stretched)
2. VERTICAL: Near parts are taller on screen (wall appears to grow)

AFFINE (wrong):                    PERSPECTIVE-CORRECT (right):
┌───┬───┬───┬───┐                  ┌─┬─┬──┬────────┐
│ 1 │ 2 │ 3 │ 4 │                  │1│2│ 3│    4   │
└───┴───┴───┴───┘                  └─┴─┴──┴────────┘
 25%  25%  25%  25%                 5% 10% 20%  65%

Problem: Equal screen space        Correct: Far parts compressed,
for unequal world distances        near parts stretched
```

However, floors and ceilings are nearly perpendicular to your view direction, so affine distortion is minimal and hard to notice. Since affine is much faster (no per-pixel division), Marathon uses it for horizontal surfaces.

### The 1/z Trick for Perspective-Correct Mapping

The key insight: while texture coordinates (u, v) don't interpolate linearly across screen space, **1/z does**.

```
Why u and v don't interpolate linearly:

World space:           Screen space:

    A─────────B          A───────────B
    │         │          │           │
    │    @    │          │  u varies │
    │ player  │          │  NON-     │
    │         │          │  linearly │

Point at world x=0.5 does NOT project to screen x=0.5!
Perspective compresses the far side.

The solution - interpolate u/z and 1/z, then divide:

    At each screen pixel:
    ┌─────────────────────────────────────────────────┐
    │  interpolated_u_over_z = lerp(u0/z0, u1/z1, t)  │
    │  interpolated_one_over_z = lerp(1/z0, 1/z1, t)  │
    │                                                 │
    │  actual_u = interpolated_u_over_z              │
    │           / interpolated_one_over_z             │
    └─────────────────────────────────────────────────┘

This division per pixel is expensive—that's why Marathon only
uses it for walls, not floors/ceilings.
```

### Let's Build: A Simple Texture Mapper

> **Important: Power-of-Two Textures**
>
> All Marathon textures must be power-of-two dimensions (64×64, 128×128, etc.). This constraint enables fast texture coordinate wrapping using bitwise AND instead of expensive modulo:
> - `u & (WIDTH - 1)` is equivalent to `u % WIDTH`
> - Only works when WIDTH is a power of two (e.g., 128 - 1 = 127 = `01111111` in binary)
> - Same trick used for angle table lookups (`angle & 511`)

**Pseudocode for affine texture mapping:**

```
function draw_textured_scanline(y, x_left, x_right,
                                 u_left, v_left, u_right, v_right,
                                 texture):
    -- Calculate deltas
    width = x_right - x_left
    if width <= 0: return

    du = (u_right - u_left) / width
    dv = (v_right - v_left) / width

    u = u_left
    v = v_left

    for x from x_left to x_right:
        -- Sample texture (wrap coordinates with mask)
        tex_x = int(u) & (TEXTURE_WIDTH - 1)
        tex_y = int(v) & (TEXTURE_HEIGHT - 1)

        color = texture[tex_y * TEXTURE_WIDTH + tex_x]
        screen[y * SCREEN_WIDTH + x] = color

        u += du
        v += dv
```

Modern C implementation:

```c
void draw_textured_scanline(
    uint32_t *framebuffer,
    int y, int x_left, int x_right,
    float u_left, float v_left,
    float u_right, float v_right,
    uint8_t *texture, int tex_width, int tex_height,
    uint32_t *palette)  // For indexed color textures
{
    int width = x_right - x_left;
    if (width <= 0) return;

    float du = (u_right - u_left) / width;
    float dv = (v_right - v_left) / width;

    float u = u_left;
    float v = v_left;

    uint32_t *dest = &framebuffer[y * SCREEN_WIDTH + x_left];

    for (int x = x_left; x < x_right; x++) {
        int tex_x = ((int)u) & (tex_width - 1);
        int tex_y = ((int)v) & (tex_height - 1);

        uint8_t texel = texture[tex_y * tex_width + tex_x];
        *dest++ = palette[texel];

        u += du;
        v += dv;
    }
}
```

**Marathon's approach:** Marathon uses fixed-point instead of float, and separates the inner loop into assembly for speed:

```c
// Marathon's inner loop structure (scottish_textures.c:107-115)
struct _horizontal_polygon_line_data {
    unsigned long source_x, source_y;     // Fixed-point texture coords
    unsigned long source_dx, source_dy;   // Fixed-point deltas
    void *shading_table;                  // Lighting lookup table
};

// The actual pixel loop (see low_level_textures.c:68-145 for C version)
while (count--) {
    uint8_t texel = texture[(source_y >> 16) * width + (source_x >> 16)];
    *dest++ = shading_table[texel];  // Apply lighting
    source_x += source_dx;
    source_y += source_dy;
}
```

> **Source:** `scottish_textures.c:107-115` for `_horizontal_polygon_line_data`, `scottish_textures.c:139-157` for `_vertical_polygon_line_data`, `low_level_textures.c:68-145` for C texture inner loops

### Detailed: Horizontal Texture Mapping (Floors/Ceilings)

Marathon's horizontal surface rendering happens in three phases:

```
Step 1: Build edge tables (Bresenham's algorithm)
        Find left/right x-coordinates for each scanline

        Polygon on screen:
              v0
             /  \
            /    \
           /      \
          v1────────v3    ← For each y-coordinate (scanline)
           \      /           we store: left_x, right_x
            \    /
             \  /
              v2

        Edge tables built:
        scanline y | left_x | right_x
        ─────────────────────────────
        100        | 320    | 320      ← Top vertex
        101        | 318    | 322      ← Expanding
        102        | 316    | 324
        ...        | ...    | ...
        150        | 200    | 440      ← Widest part
        ...        | ...    | ...
        200        | 320    | 320      ← Bottom vertex

Step 2: Precalculate texture coordinates per scanline
        For each scanline:
          source_x = (dhcosine - screen_x×hsine)/screen_y + origin.x
          source_dx = -hsine/screen_y    (increment per pixel)
          source_y = (screen_x×hcosine + dhsine)/screen_y + origin.y
          source_dy = hcosine/screen_y   (increment per pixel)

Step 3: Rasterize each scanline
        For scanline at y=150:

        Screen space:
        x: 200             320             440
           |───────────────|───────────────|
           left_x       (middle)        right_x

        Texture space (64×64 floor texture):
           u: 10           32              54
           v: 15           15              15  (constant for scanline)

        For each pixel from left_x to right_x:
          texture_pixel = texture[source_y >> 16][source_x >> 16]
          screen_pixel = shading_table[texture_pixel]
          *screen++ = screen_pixel
          source_x += source_dx
          source_y += source_dy

Why affine works for floors:
  - Viewed at shallow angles, distortion is minimal
  - Floor is perpendicular to view (less perspective needed)
  - MUCH faster than perspective-correct (no per-pixel divide)
```

### Detailed: Vertical Texture Mapping (Walls)

Walls use column-based rendering with perspective correction:

```
Step 1: Build column tables (y-coordinates for each screen column)
        Find top/bottom y-coordinates for each vertical strip

        Wall polygon on screen:

        y=100  ┌─────────────────┐  ← ceiling edge (top of wall)
               │  /           \  │
               │ /             \ │
               │/               \│
        y=300  └─────────────────┘  ← floor edge (bottom of wall)

               x=200    ...    x=400

        Column tables built:
        screen_x | top_y | bottom_y | world_x | shading_table
        ───────────────────────────────────────────────────────
        200      | 100   | 300      | 512     | table[distant]
        201      | 102   | 298      | 510     | table[distant]
        ...      | ...   | ...      | ...     | ...
        300      | 120   | 280      | 256     | table[close]
        ...      | ...   | ...      | ...     | ...
        400      | 100   | 300      | 512     | table[distant]

Step 2: Precalculate texture coordinates per column
        For each column at screen_x:

          // Texture horizontal position (which column of wall texture)
          tx = origin.y + (screen_x / view.world_to_screen_x) × vector.j
          texture_column_index = tx & (TEXTURE_WIDTH - 1)

          // Texture vertical position (perspective-correct)
          world_x = origin.x + tx × vector.i  // Distance to wall
          ty = (world_x × screen_y - origin.z) / vector.k
          ty_delta = -world_x / (vector.k >> 8)  // Per-pixel increment

Step 3: Rasterize column-by-column
        For column at x=300:

        Screen space (vertical strip):
        y: 120  ─  ← top_y
           140  │
           160  │  Draw from top to bottom
           180  │
           200  │
           240  │
           260  │
           280  ─  ← bottom_y

        Texture space (128-pixel tall wall texture):
        Column 45 of texture:
           v:  0  ─  ← Top of texture
              16  │
              32  │
              48  │
              64  │
              80  │
              96  │
             112  ─  ← Bottom of texture (wraps if needed)

        Inner loop (for y from top_y to bottom_y):
          texture_v = (texture_y >> VERTICAL_TEXTURE_FREE_BITS) & mask
          pixel = texture_column[texture_v]
          screen[y × width + x] = shading_table[pixel]
          texture_y += texture_dy  // Perspective-correct increment

Why perspective-correct is necessary for walls:
  - Walls are viewed at steep angles (parallel to view direction)
  - Linear interpolation causes visible distortion
  - Each column represents same world distance
  - Texture vertical coordinate must account for perspective
```

**Column-Based Wall Rendering:**

```
Wall rendering proceeds column-by-column (vertical strips):

Texture (128×128 wall):          Screen columns:           Rendering order:
     0    45   90   127               200  300  400
     ↓     ↓    ↓     ↓                 ↓    ↓    ↓
   ┌─────────────────┐             ┌───┬───┬───┬───┐         1. Column 200
   │█░░░░░░░░░░░░░░░█│             │   │   │   │   │            (texture col 0)
   │█░░░BRICK░░░░░░░█│             │   │   │   │   │
   │█░░TEXTURE░░░░░░█│  ────────>  │ T │ T │ T │ T │         2. Column 201
   │█░░░░░░░░░░░░░░░█│             │ E │ E │ E │ E │            (texture col 1)
   │█░░░░░░░░░░░░░░░█│             │ X │ X │ X │ X │
   │█████████████████│             │ T │ T │ T │ T │         ...
   └─────────────────┘             └───┴───┴───┴───┘

   Each screen column uses ONE texture column
   Vertical position calculated with perspective divide
   Much faster than per-pixel perspective (only per-column)

Example: Rendering screen column x=300
  1. Look up: texture_column_index = 45 (from precalculation)
  2. Get texture column: texture_column = texture->row_addresses[45]
  3. Loop y from 120 to 280:
     - Calculate v-coordinate (with perspective)
     - Fetch: pixel = texture_column[v]
     - Shade: output = shading_table[pixel]
     - Write: screen[y × 640 + 300] = output
```

### Perspective-Correct Mapping for Walls

For walls, we need perspective-correct mapping. The key insight is that while texture coordinates don't interpolate linearly in screen space, **1/z** does.

**The 1/z trick:**

```
Instead of interpolating u and v directly:
    u_screen = lerp(u_left, u_right, t)  // WRONG for perspective

Interpolate u/z and v/z, then divide by 1/z:
    u_over_z = lerp(u_left/z_left, u_right/z_right, t)
    one_over_z = lerp(1/z_left, 1/z_right, t)
    u_screen = u_over_z / one_over_z  // CORRECT
```

Marathon simplifies this by rendering walls **column by column** (vertical strips) instead of scanline by scanline. Each column has constant depth, so only vertical texture coordinate needs the perspective correction.

```c
// For each column x:
void draw_wall_column(int x, int y_top, int y_bottom,
                      int texture_column,
                      float v_top, float v_delta,
                      uint8_t *texture, uint32_t *shading_table) {

    float v = v_top;
    uint32_t *dest = &framebuffer[y_top * SCREEN_WIDTH + x];

    for (int y = y_top; y < y_bottom; y++) {
        int tex_v = ((int)v) & (TEXTURE_HEIGHT - 1);
        uint8_t texel = texture[tex_v * TEXTURE_WIDTH + texture_column];
        *dest = shading_table[texel];
        dest += SCREEN_WIDTH;  // Move down one row
        v += v_delta;
    }
}
```

The `v_delta` is calculated per-column to account for perspective.

**Why world_x (distance) matters for perspective:**

```
        Top view of wall:

        Camera                   Wall (vertical in 3D)
          @                      │
           \                     │
            \  view ray          │
             \                   │
              \                  │
               \                 │
                \                │
                 \               │
                  \              │
                   \             │
                    \            │
                     \           │
                      \          │
                       \         │
                        \        │
                         \       │
                          \      │
                           \     │
                            \    │
                             \   │
                              \  │
                               \ │
                                \│
                              Hit point
                              world_x = distance

        Problem: Screen pixels are evenly spaced, but represent
                 different world heights at different distances

        Far part of wall:
          screen_y = 10 pixels  →  world_height = 100 units
          (wall is distant, appears small)

        Near part of wall:
          screen_y = 10 pixels  →  world_height = 20 units
          (wall is close, appears large)

        Solution: Marathon calculates per-column:
          ty_delta = -world_x / (vector.k >> 8)

          Closer walls (small world_x) → smaller ty_delta → slower texture scroll
          Farther walls (large world_x) → larger ty_delta → faster texture scroll

          Result: Perspective-correct texture mapping!
```

### Lighting with Shading Tables

Marathon uses **pre-computed shading tables** for lighting. Instead of calculating lighting per pixel, it creates 256 tables (one per light level), each mapping the 256 palette colors to their shaded versions.

```
shading_tables[light_level][original_color] = shaded_color

Example:
  Full brightness:  shading_tables[255][RED] = BRIGHT_RED
  Half brightness:  shading_tables[128][RED] = MEDIUM_RED
  Darkness:         shading_tables[32][RED]  = DARK_RED
```

This turns lighting into a single table lookup per pixel—extremely fast.

**Shading Tables:**
- 8-bit mode: 32 tables × 256 colors = 8KB total
- 16-bit/32-bit mode: 64 tables × 256 entries
- Each entry maps palette index → lit pixel value

### Edge Table Building (Bresenham's DDA Algorithm)

Marathon uses a modified Bresenham line algorithm to build **edge tables**—arrays of x or y coordinates for each scanline or column of a polygon.

**Why Edge Tables?**

Instead of computing polygon edges during rasterization, Marathon precomputes all edge coordinates into "scratch tables" before drawing. This separates the geometric work from the pixel-writing inner loop.

```c
// Scratch table architecture (scottish_textures.c)
#define MAXIMUM_SCRATCH_TABLE_ENTRIES 1024

static short *scratch_table0;  // Left edges (horizontal) or top edges (vertical)
static short *scratch_table1;  // Right edges (horizontal) or bottom edges (vertical)
```

**build_x_table()** - For horizontal surfaces (floors/ceilings):

```
Given edge from (x0,y0) to (x1,y1), builds table of x values for each y:

    (x0,y0)                           table[0] = x0
       *                              table[1] = x0+dx
      / \                             table[2] = x0+2dx
     /   \                            ...
    /     \                           table[n] = x1
   *-------*
  (x1,y1)

Algorithm (Bresenham's DDA):
┌──────────────────────────────────────────────────────────────────┐
│  dx = x1 - x0;  adx = |dx|;  dx = SGN(dx)                        │
│  dy = y1 - y0;  ady = |dy|;  dy = SGN(dy)                        │
│                                                                  │
│  if (adx >= ady)  // X-dominant line                             │
│  {                                                               │
│      d = adx - ady                                               │
│      delta_d = -2 * ady                                          │
│      d_max = 2 * adx                                             │
│                                                                  │
│      while (adx-- >= 0)                                          │
│      {                                                           │
│          if (d < 0)                                              │
│          {                                                       │
│              y += 1                    // Step in y              │
│              d += d_max                                          │
│              *record++ = x             // Record x at this y     │
│          }                                                       │
│          x += dx                       // Always step in x       │
│          d += delta_d                                            │
│      }                                                           │
│  }                                                               │
│  else  // Y-dominant line                                        │
│  {                                                               │
│      // Record x for EVERY y step                                │
│      while (ady-- >= 0)                                          │
│      {                                                           │
│          if (d < 0) x += dx, d += d_max                          │
│          *record++ = x                 // Record every iteration │
│          y += 1, d += delta_d                                    │
│      }                                                           │
│  }                                                               │
└──────────────────────────────────────────────────────────────────┘
```

**Edge Table Usage in Polygon Rasterization:**

```
Horizontal Polygon (floor/ceiling):

Step 1: Find highest and lowest vertices
        highest_vertex = v0 (smallest y)
        lowest_vertex = v2 (largest y)

Step 2: Walk left edge (counterclockwise) building left_table
        Walk right edge (clockwise) building right_table

        Screen Y    left_table    right_table
        ─────────────────────────────────────
        100         320           320         ← Top vertex
        110         300           340
        120         280           360
        ...         ...           ...
        200         200           440         ← Bottom

Step 3: For each scanline y from 100 to 200:
        left_x = left_table[y - 100]
        right_x = right_table[y - 100]
        → Rasterize pixels from left_x to right_x

Vertical Polygon (wall):

Step 1: Find leftmost and rightmost vertices
        leftmost = v0 (smallest x)
        rightmost = v2 (largest x)

Step 2: Walk top edge building top_table
        Walk bottom edge building bottom_table

        Screen X    top_table    bottom_table
        ─────────────────────────────────────
        200         100          300          ← Left edge
        250         120          280
        300         140          260
        ...         ...          ...
        400         100          300          ← Right edge

Step 3: For each column x from 200 to 400:
        top_y = top_table[x - 200]
        bottom_y = bottom_table[x - 200]
        → Rasterize pixels from top_y to bottom_y
```

> **Source:** `scottish_textures.c:1176-1290` for `build_x_table()` and `build_y_table()`

### Span Caching / Precalculation System

Marathon's key optimization is **precalculating texture mapping parameters** for entire scanlines or columns before the pixel-writing loop. This is stored in the `precalculation_table`.

**Precalculation Table:**

```c
#define MAXIMUM_PRECALCULATION_TABLE_ENTRY_SIZE 34  // bytes per entry
static void *precalculation_table;  // Sized for MAXIMUM_SCRATCH_TABLE_ENTRIES
```

**Horizontal Polygon Line Data** (for floors/ceilings):

```c
struct _horizontal_polygon_line_data {
    unsigned long source_x, source_y;     // Starting texture coords (16.16 fixed)
    unsigned long source_dx, source_dy;   // Delta per pixel (16.16 fixed)
    void *shading_table;                  // Pre-selected lighting table
};
```

**Vertical Polygon Line Data** (for walls):

```c
struct _vertical_polygon_data {
    short downshift;   // Bit shift for texture lookup
    short x0;          // Starting screen x
    short width;       // Number of columns
};

struct _vertical_polygon_line_data {
    void *shading_table;           // Pre-selected lighting table
    pixel8 *texture;               // Pointer to texture column data
    long texture_y, texture_dy;    // Starting v and delta (fixed-point)
};
```

**Precalculation Flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                  PRECALCULATION PHASE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For HORIZONTAL surface (floor/ceiling):                        │
│                                                                 │
│    _pretexture_horizontal_polygon_lines():                      │
│      For each scanline y:                                       │
│        1. Calculate view-space transform                        │
│        2. Compute texture start (source_x, source_y)            │
│        3. Compute texture delta (source_dx, source_dy)          │
│        4. Select shading table based on depth                   │
│        5. Store in precalculation_table[y]                      │
│                                                                 │
│  For VERTICAL surface (wall):                                   │
│                                                                 │
│    _pretexture_vertical_polygon_lines():                        │
│      For each column x:                                         │
│        1. Calculate wall distance (world_x)                     │
│        2. Compute texture column index                          │
│        3. Compute texture_y start position                      │
│        4. Compute texture_dy (perspective-correct delta)        │
│        5. Get pointer to texture column                         │
│        6. Select shading table based on world_x                 │
│        7. Store in precalculation_table[x]                      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                  RASTERIZATION PHASE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  _texture_horizontal_polygon_lines8/16/32():                    │
│    For each scanline:                                           │
│      Load precalculated data from table                         │
│      For each pixel:                                            │
│        texture_pixel = texture[source_y >> 16][source_x >> 16]  │
│        screen_pixel = shading_table[texture_pixel]              │
│        *screen++ = screen_pixel                                 │
│        source_x += source_dx                                    │
│        source_y += source_dy                                    │
│                                                                 │
│  _texture_vertical_polygon_lines8/16/32():                      │
│    For each column:                                             │
│      Load precalculated data from table                         │
│      For each pixel:                                            │
│        v = (texture_y >> DOWNSHIFT) & mask                      │
│        texture_pixel = texture_column[v]                        │
│        screen_pixel = shading_table[texture_pixel]              │
│        screen[y * stride + x] = screen_pixel                    │
│        texture_y += texture_dy                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Why This Matters:**
- Inner loop contains NO divides (perspective math done in precalc)
- Shading table lookup is single array index
- All branching decisions made before rasterization
- Cache-friendly memory access pattern

> **Source:** `scottish_textures.c:1037-1125` for `_pretexture_horizontal_polygon_lines()`, `scottish_textures.c:930-1035` for `_pretexture_vertical_polygon_lines()`

### Source Reference

Texture mapping lives in `scottish_textures.c`:

| Function | Line | Purpose |
|----------|------|---------|
| `texture_horizontal_polygon()` | 277 | Affine floor/ceiling mapper |
| `texture_vertical_polygon()` | 476 | Perspective-correct wall mapper |
| `_pretexture_horizontal_polygon_lines()` | 1041 | Setup horizontal texture coords |
| `struct _horizontal_polygon_line_data` | 107 | Horizontal texture line data |
| `struct _vertical_polygon_line_data` | 139 | Vertical texture line data |

C fallback implementations in `low_level_textures.c`:

| Function | Line | Purpose |
|----------|------|---------|
| `_texture_horizontal_polygon_lines8()` | 68 | 8-bit horizontal inner loop |
| `_texture_vertical_polygon_lines8()` | 226 | 8-bit vertical inner loop |

---

## 5.7 The Complete Rendering Pipeline

Now let's see how all the pieces fit together in Marathon's actual `render_view()` function.

### Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                      render_view() PIPELINE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. UPDATE CAMERA                                                    │
│     update_view_data() - Set up projection matrices                  │
│                                                                      │
│  2. BUILD VISIBILITY                                                 │
│     build_render_tree() - Ray cast to find visible polygons          │
│                                                                      │
│  3. SORT FOR RENDERING                                               │
│     sort_render_tree() - Convert tree to back-to-front order         │
│                                                                      │
│  4. COLLECT OBJECTS                                                  │
│     build_render_object_list() - Find visible sprites/monsters       │
│                                                                      │
│  5. RENDER POLYGONS                                                  │
│     render_tree() - For each polygon back-to-front:                  │
│       ├─ Render ceiling (if camera below ceiling)                    │
│       ├─ Render walls/sides                                          │
│       ├─ Render interior objects (monsters, items in this polygon)   │
│       └─ Render floor (if camera above floor)                        │
│                                                                      │
│  6. RENDER OVERLAY                                                   │
│     render_viewer_sprite_layer() - Weapon sprite, HUD overlays       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Back-to-Front Rendering (Painter's Algorithm)

Marathon draws polygons from farthest to nearest. This means nearer surfaces automatically overdraw farther ones—no Z-buffer needed.

```
Rendering order example:

Step 1: Draw far room (Polygon C)
        ┌──────────────────────────────────────┐
        │                                      │
        │          C C C C C C C               │
        │          C C C C C C C               │
        │                                      │
        └──────────────────────────────────────┘

Step 2: Draw middle room (Polygon B) - overwrites parts of C
        ┌──────────────────────────────────────┐
        │                                      │
        │      B B B B C C C C C               │
        │      B B B B C C C C C               │
        │                                      │
        └──────────────────────────────────────┘

Step 3: Draw near room (Polygon A) - overwrites parts of B
        ┌──────────────────────────────────────┐
        │                                      │
        │  A A A A A B C C C C C               │
        │  A A A A A B C C C C C               │
        │                                      │
        └──────────────────────────────────────┘

Result: Correct depth ordering without Z-buffer!
```

**Per-Scanline Processing within a Polygon:**

For each polygon, the rasterizer works scanline-by-scanline:

```
Screen Y-coordinate
     0  ┌────────────────────────────────┐
     10 │     ┌──────────────┐           │  ← Ceiling starts
     50 │     │  Ceiling     │           │
    100 │     └──────────────┘           │  ← Ceiling ends
    150 │     ┌──────────────┐           │  ← Upper wall
    200 │     │    Wall      │           │
    300 │     │              │           │
    350 │     └──────────────┘           │  ← Floor starts
    400 │     ┌──────────────┐           │
    450 │     │    Floor     │           │
    479 └─────┴──────────────┴───────────┘

Per scanline (e.g., y=200):
1. Check polygon's clipping window: y in [y0, y1]?
2. Interpolate left/right x coordinates for this y
3. For each x in [left_x, right_x]:
   - Calculate texture coordinates
   - Lookup shading table
   - Write pixel to framebuffer[y * width + x]
```

### Object Rendering

Sprites (monsters, items, projectiles) are rendered with their containing polygon. Marathon tracks which polygon each object is in and renders it during that polygon's pass.

Objects that span multiple polygons ("exterior objects") are handled specially—they're drawn with a combined clipping window from all polygons they intersect.

### Overdraw and the Painter's Algorithm

**Yes, Marathon has overdraw.** Without a Z-buffer (too expensive for 1994 hardware), Marathon uses the painter's algorithm: draw far surfaces first, then let nearer surfaces paint over them.

**What gets overdrawn:**

```
Far polygon drawn first:       Near polygon overwrites:
┌─────────────────────┐        ┌─────────────────────┐
│ C C C C C C C C C C │   →    │ A A A A A C C C C C │
│ C C C C C C C C C C │        │ A A A A A C C C C C │
│ C C C C C C C C C C │        │ A A A A A C C C C C │
└─────────────────────┘        └─────────────────────┘
    Pixels wasted                 Some C pixels were
    drawing area                  drawn then overwritten
    behind A                      (overdraw)
```

**Render order within each polygon** (from `render_tree()` in render.c:2188-2337):

1. **Ceiling** (if above viewer's eye height)
2. **Walls/sides** (visible ones only, via `_side_is_visible` flag)
3. **Floor** (if below viewer's eye height)
4. **Exterior objects** (sprites spanning from other polygons)

From Jason Jones' comments in render.c (October 1994):
> "in order to correctly handle objects below the viewer projecting into higher polygons we need to sort objects inside nodes (to be drawn after their walls and ceilings but before their floors)"

**What minimizes overdraw:**

1. **Portal clipping windows** - Each polygon only renders within its screen-space bounds (`x0, x1, y0, y1` in `clipping_window_data`). A polygon seen through a narrow doorway only draws that narrow slice.

2. **Visibility culling** - Only polygons reachable through visible portals are rendered at all. Typical frame: 50-100 polygons from 500-1000 total.

3. **Surface culling** - Floors above the viewer and ceilings below the viewer are skipped entirely.

4. **Side visibility flags** - Walls facing away from the viewer aren't drawn (`TEST_RENDER_FLAG(side_index, _side_is_visible)`).

**Why no Z-buffer?**

A software Z-buffer requires a depth comparison and conditional write per pixel:
```c
// Z-buffer approach (NOT used by Marathon)
if (depth < zbuffer[pixel]) {
    zbuffer[pixel] = depth;
    framebuffer[pixel] = color;
}
```

On a 25 MHz 68040, this comparison per pixel was too expensive. The painter's algorithm trades some overdraw for simpler per-pixel logic (unconditional write).

> **Source:** `render.c:2182-2341` for `render_tree()`, `render.c:1-54` for Jason Jones' historical comments on render order challenges

### Render Tree Architecture

Marathon builds two tree structures during rendering: the **node tree** (raw portal traversal) and the **sorted node tree** (depth-ordered for rendering).

**Node Tree** (portal traversal result):

```c
#define MAXIMUM_NODES 512

struct node_data {
    word flags;
    short polygon_index;

    // Clipping data accumulated from portal crossings
    short clipping_endpoint_count;
    short clipping_endpoints[MAXIMUM_CLIPPING_ENDPOINTS_PER_NODE];  // 4 max
    short clipping_line_count;
    short clipping_lines[MAXIMUM_CLIPPING_LINES_PER_NODE];          // 6 max

    // Tree linkage
    struct node_data *parent;     // Parent node (NULL for root)
    struct node_data **reference; // Pointer to our parent's child slot
    struct node_data *siblings;   // Next sibling in parent's child list
    struct node_data *children;   // First child node
};
```

**Sorted Node Tree** (rendering order):

```c
#define MAXIMUM_SORTED_NODES 128

struct sorted_node_data {
    short polygon_index;

    struct render_object_data *interior_objects;  // Objects inside polygon
    struct render_object_data *exterior_objects;  // Objects overlapping from outside

    struct clipping_window_data *clipping_windows;  // Combined clip regions
};
```

**Tree Building Process:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    build_render_tree()                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Initialize root node with player's polygon                  │
│     nodes[0] = {polygon_index: player_polygon, parent: NULL}    │
│                                                                 │
│  2. Cast rays at screen edges:                                  │
│     cast_render_ray(left_edge, counterclockwise_bias)           │
│     cast_render_ray(right_edge, clockwise_bias)                 │
│                                                                 │
│  3. Process polygon queue (BFS):                                │
│     while (polygon_queue not empty):                            │
│       polygon = dequeue()                                       │
│       for each vertex in polygon:                               │
│         if not visited:                                         │
│           transform to view space                               │
│           calculate screen x-coordinate                         │
│           if within view cone:                                  │
│             cast_render_ray(vertex_vector)                      │
│           mark visited                                          │
│                                                                 │
│  4. cast_render_ray() traces through polygon graph:             │
│     Find which polygon edge ray crosses                         │
│     Get adjacent polygon through that edge                      │
│     Create/update child node                                    │
│     Accumulate clipping data                                    │
│     Continue until ray exits map                                │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                     sort_render_tree()                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Traverse node tree depth-first (farthest first)             │
│                                                                 │
│  2. For each node, create sorted_node with:                     │
│     - Polygon index                                             │
│     - Combined clipping windows from all paths                  │
│     - Object lists (filled later)                               │
│                                                                 │
│  3. Result: Array of sorted_nodes in back-to-front order        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Node Tree vs Sorted Tree:**

```
Node Tree (graph, may have multiple paths to same polygon):

              Root (P0)
             /    |    \
         N1(P1) N2(P2) N3(P3)
           |      |
         N4(P4) N5(P4)  ← Same polygon, different paths!

Sorted Tree (linear array, each polygon once):

  sorted_nodes[] = [P4, P2, P1, P3, P0]
                    ↑              ↑
                farthest      nearest (player)

  Clipping windows combined from all paths to each polygon
```

### Object Depth Sorting

Marathon's object sorting is complex due to edge cases with overlapping objects in multiple polygons.

**Object Sorting Challenges** (from render.c comments):
1. Objects can overlap into polygons clipped behind them
2. Multiple non-overlapping objects with uncertain relative order
3. Objects below viewer projecting into higher polygons
4. Parasitic objects (players with attached items)

**Render Object Structure:**

```c
#define MAXIMUM_RENDER_OBJECTS 72

struct render_object_data {
    struct sorted_node_data *node;           // Polygon we're drawn in
    struct clipping_window_data *clipping_windows;  // Our clipping region
    struct render_object_data *next_object;  // Linked list
    struct rectangle_definition rectangle;   // Screen bounds + texture
    short ymedia;                            // Media clipping boundary
};
```

**Object Sorting Algorithm:**

```
┌─────────────────────────────────────────────────────────────────┐
│               build_render_object_list()                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  For each sorted polygon (back to front):                       │
│    For each object in polygon's object list:                    │
│                                                                 │
│      1. build_render_object():                                  │
│         - Transform object origin to view space                 │
│         - Calculate screen rectangle bounds                     │
│         - Determine which polygons object overlaps              │
│                                                                 │
│      2. build_base_node_list():                                 │
│         - Find all sorted nodes the object spans                │
│         - Based on object's left/right screen edges             │
│         - Returns array of base_nodes                           │
│                                                                 │
│      3. sort_render_object_into_tree():                         │
│         For each base_node:                                     │
│           Insert object into node's interior or exterior list   │
│           based on depth comparison                             │
│                                                                 │
│      4. build_aggregate_render_object_clipping_window():        │
│         - Combine clipping windows from all base nodes          │
│         - Object is clipped by intersection of all              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Interior vs Exterior Objects:                                  │
│                                                                 │
│  Interior: Object's origin polygon == node's polygon            │
│    → Drawn AFTER walls, BEFORE floor (standard depth order)     │
│                                                                 │
│  Exterior: Object overlaps into node from another polygon       │
│    → Requires special handling for correct occlusion            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Depth Ordering Within Node:**

```
For each polygon node, objects are drawn in this order:

  1. Ceiling (if viewer below ceiling)
  2. Walls (back to front within polygon)
  3. Interior objects (depth sorted) ← Objects whose origin is HERE
  4. Exterior objects (depth sorted) ← Objects overlapping FROM elsewhere
  5. Floor (if viewer above floor)

This ensures:
- Objects behind walls are occluded
- Objects in front of floors are visible
- Overlapping objects sorted correctly
```

> **Source:** `render.c:1674-1850` for `build_render_object_list()`, `render.c:1850-1950` for object sorting

### Polygon Clipping System

Marathon uses a hierarchical clipping system with endpoint clips, line clips, and clipping windows.

**Clipping Data Structures:**

```c
#define MAXIMUM_ENDPOINT_CLIPS 64
#define MAXIMUM_LINE_CLIPS 256
#define MAXIMUM_CLIPPING_WINDOWS 256

struct endpoint_clip_data {
    word flags;              // _clip_left, _clip_right
    short x;                 // Screen x-coordinate
    world_vector2d vector;   // View-space direction to endpoint
};

struct line_clip_data {
    word flags;              // _clip_up, _clip_down
    short x0, x1;            // Screen x range
    world_vector2d top_vector, bottom_vector;  // View-space clip planes
    short top_y, bottom_y;   // Screen y range
};

struct clipping_window_data {
    world_vector2d left, right, top, bottom;  // Clip plane normals
    short x0, x1, y0, y1;                     // Screen bounds
    struct clipping_window_data *next_window; // Linked list
};
```

**Clip Flags:**

```c
enum {
    _clip_left  = 0x0001,  // Clip against left edge
    _clip_right = 0x0002,  // Clip against right edge
    _clip_up    = 0x0004,  // Clip against top edge
    _clip_down  = 0x0008   // Clip against bottom edge
};
```

**Clipping Algorithm:**

```
┌─────────────────────────────────────────────────────────────────┐
│              Polygon Clipping Pipeline                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  xy_clip_horizontal_polygon():                                  │
│    Clips 2D polygon against vertical clip plane                 │
│    Used for: Left/right screen edge clipping                    │
│                                                                 │
│    For each edge of polygon:                                    │
│      Compute: cross = point × clip_vector                       │
│      If points on opposite sides of clip plane:                 │
│        Interpolate new vertex at intersection                   │
│        Set appropriate clip flag on new vertex                  │
│                                                                 │
│  z_clip_horizontal_polygon():                                   │
│    Clips against horizontal plane at height z                   │
│    Used for: Floor/ceiling height clipping                      │
│                                                                 │
│  xz_clip_vertical_polygon():                                    │
│    Clips 3D polygon for wall rendering                          │
│    Handles both vertical and depth clipping                     │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│              Clipping Window Accumulation                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  As render tree is built, each portal crossing adds clipping:   │
│                                                                 │
│    Parent polygon         Child polygon (through portal)        │
│    ┌──────────────┐       ┌──────────────┐                      │
│    │              │       │              │                      │
│    │    ┌────┐    │       │    Content   │                      │
│    │    │ P  │────┼───────┼──> clipped   │                      │
│    │    │ O  │    │       │    by portal │                      │
│    │    │ R  │    │       │    edges     │                      │
│    │    │ T  │    │       │              │                      │
│    │    │ A  │    │       │              │                      │
│    │    │ L  │    │       │              │                      │
│    │    └────┘    │       └──────────────┘                      │
│    └──────────────┘                                             │
│                                                                 │
│  Child inherits parent's clip window PLUS portal edges          │
│  Result: Nested clipping for arbitrary portal depth             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Source:** `render.c:2606-3100` for clipping functions, `render.c:207-213` for `struct clipping_window_data`

### Transfer Modes (Special Effects)

Marathon uses transfer modes for special visual effects on textures and sprites.

**Effect Types:**

```c
_textured_transfer        // Normal texture mapping
_tinted_transfer         // Color overlay (tinting)
_solid_transfer          // Solid color fill
_shadeless_transfer      // Ignores lighting
_static_transfer         // TV snow/static effect
_big_landscaped_transfer // Screen-space texture (skybox)
```

**Special Effects Examples:**
- `_xfer_invisibility` - Semi-transparent via shading table manipulation
- `_xfer_fold_in/fold_out` - Teleport shrink/grow animation
- `_xfer_fast_horizontal_slide` - Scrolling texture (conveyor belts, water)
- `_xfer_wobble` - Perspective distortion effect
- `_xfer_pulsate` - Breathing/pulsing scale effect

**How Texture Scrolling Works:**

The slide transfer modes (`_xfer_horizontal_slide`, `_xfer_fast_horizontal_slide`, etc.) animate textures by adding a time-based offset to the texture origin:

```c
// Each tick, the texture origin shifts:
texture_origin.x += scroll_speed;  // Horizontal slide
texture_origin.y += scroll_speed;  // Vertical slide

// The offset wraps using the power-of-two trick:
tex_u = (base_u + texture_origin.x) & (TEXTURE_WIDTH - 1);
```

This creates flowing water, conveyor belts, and other animated surface effects without needing animated texture frames.

Transfer modes are applied during texture mapping by selecting different shading tables or pixel-writing routines.

> **Source:** `render.c:3390-3500` for transfer mode handling, `scottish_textures.h:15-30` for transfer mode enum

### Landscape Rendering (Sky/Outdoor Areas)

Marathon's landscape rendering creates the illusion of outdoor environments and distant scenery. Unlike normal textures that are mapped to world geometry, **landscape textures are anchored in screen space**—they don't move with perspective as you walk around, only as you rotate your view.

**How Landscape Differs from Normal Textures:**

```
NORMAL TEXTURE MAPPING:                  LANDSCAPE TEXTURE MAPPING:

World space → Screen space               Screen space directly

  Wall moves closer                        Sky stays fixed in place
  as you walk toward it                    only rotates as you turn

     ┌─────┐  →  ┌───────────┐            ┌─────────────────────────┐
     │wall │     │  wall     │            │         SKY             │
     │     │     │           │            │                         │
     └─────┘     └───────────┘            └─────────────────────────┘
     (far)         (near)                  (same regardless of position)

Texture scales with distance              Texture size is constant
```

**The `_big_landscaped_transfer` Mode:**

When a polygon has the `_xfer_landscape` transfer mode, it uses screen-space texture mapping:

```c
// From scottish_textures.h:23
_big_landscaped_transfer  // "does not distort texture (texture is anchored in screen-space)"
```

**Landscape Texture Coordinate Calculation:**

The key insight is that landscape textures map screen coordinates directly to texture coordinates, with the player's yaw angle controlling horizontal offset:

```c
// From scottish_textures.c:1139-1166 (_prelandscape_horizontal_polygon_lines)

// Horizontal position based on player's yaw angle
first_pixel = view->yaw << (landscape_width_bits + LANDSCAPE_REPEAT_BITS + FIXED_FRACTIONAL_BITS - ANGULAR_BITS);

// Each screen pixel advances by this amount in texture space
pixel_delta = (view->half_cone << (1 + landscape_width_bits + ...)) / view->standard_screen_width;

// For each scanline:
data->source_x = (first_pixel + x0 * pixel_delta) << landscape_free_bits;
data->source_dx = pixel_delta << landscape_free_bits;

// Vertical position based on screen Y (centered at horizon)
data->source_y = texture_height - PIN(y0 * pixel_delta + texture_height/2, 0, texture_height-1) - 1;
```

**Visual Explanation:**

```
                        Player looking at landscape

Screen Y    Texture V
   0  ┌─────────────────────────────┐  texture_height (top of sky)
      │                             │
  120 │         SKY TEXTURE         │  texture_height/2 (horizon)
      │                             │
  240 │                             │  0 (bottom of sky texture)
      ├─────────────────────────────┤
  280 │       (ground below)        │
  480 └─────────────────────────────┘

      ←───── Screen X ─────────────→
            ↓ maps to ↓
      ←─── Texture U (based on yaw) ───→

As player ROTATES (yaw changes):
  - Texture scrolls horizontally (sky rotates)
  - first_pixel changes based on view->yaw

As player MOVES (position changes):
  - Texture does NOT move (sky stays fixed)
  - Only world geometry changes
```

**When Landscape Mode is Used:**

1. **Line flag**: Lines can be marked with `LANDSCAPE_LINE_BIT` (map.h:393)
2. **Side transfer mode**: Sides can have `_xfer_landscape` transfer mode
3. **Polygon transfer mode**: Floor/ceiling can use landscape mode

```c
// From map.c:932-973 - check if a line's side uses landscape mode
boolean line_is_landscaped(short polygon_index, short line_index, world_distance z)
{
    // Checks side type and transfer mode at the given height
    // Returns TRUE if the appropriate texture uses _xfer_landscape
}
```

**Landscape Texture Dimensions:**

Landscape textures are typically 512 or 1024 pixels wide (matching Marathon's 512-entry angle table for smooth rotation):

```c
// From scottish_textures.c:1136
short landscape_width_bits = polygon->texture->height == 1024 ? 10 : 9;
```

The width must be power-of-two for the bitwise wrap-around trick:
```c
// Texture lookup with automatic wrap
pixel = read[source_x >> landscape_texture_width_downshift];
// where landscape_texture_width_downshift = 32 - landscape_width_bits
```

**Rendering Pipeline for Landscapes:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Landscape Rendering Flow                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Polygon has _xfer_landscape transfer mode                   │
│     ↓                                                           │
│  2. render.c converts to _big_landscaped_transfer (line 3490)   │
│     ↓                                                           │
│  3. texture_horizontal_polygon() dispatches to landscape path   │
│     ↓                                                           │
│  4. _prelandscape_horizontal_polygon_lines() precalculates:     │
│     - source_x from player yaw + screen x                       │
│     - source_dx as constant pixel delta                         │
│     - source_y from screen y (centered at horizon)              │
│     ↓                                                           │
│  5. _landscape_horizontal_polygon_lines8/16/32() renders:       │
│     - Reads texture row at source_y                             │
│     - For each screen pixel: texture[source_x >> downshift]     │
│     - source_x += source_dx                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Source:** `scottish_textures.c:1124-1173` for `_prelandscape_horizontal_polygon_lines()`, `low_level_textures.c:143-210` for `LANDSCAPE_HORIZONTAL_POLYGON_LINES()` inner loop

### Source Reference

The main render loop in `render.c`:

```c
// render.c:497-540 (simplified)
void render_view(
    struct view_data *view,
    struct bitmap_definition *destination)
{
    // 1. Update camera matrices (render.c:561)
    update_view_data(view);

    // 2. Build visibility tree (render.c:702)
    initialize_render_tree();
    build_render_tree(view);

    // 3. Sort for back-to-front rendering (render.c:1116)
    sort_render_tree();

    // 4. Find visible objects (render.c:1674)
    build_render_object_list();

    // 5. Render all polygons (render.c:2182)
    render_tree(view, destination);

    // 6. Draw weapon overlay (render.c:3505)
    render_viewer_sprite_layer(view, destination);
}
```

---

## 5.8 Summary

Marathon's rendering system solves the classic problem of turning a 3D world into a 2D image efficiently. The key techniques are:

**Portal-Based Visibility:**
- Start from player's room, recursively discover visible rooms through portals
- Each portal narrows the visible region
- Build a tree structure for proper render ordering

**Two-Pass Texture Mapping:**
- Affine mapping for floors/ceilings (fast, acceptable distortion)
- Perspective-correct mapping for walls (slower, accurate)

**Optimizations:**
- Fixed-point math for speed and determinism
- Pre-computed shading tables for lighting
- Edge tables (Bresenham) for polygon rasterization
- Assembly inner loops for the tightest code paths

**Performance Breakdown (typical frame):**
```
Portal culling:    500 polygons → 50 visible (10× reduction)
Edge building:     50 polygons × ~100 edges = ~5,000 edge calculations
Texture precalc:   ~5,000 lines × 2 coordinates = ~10,000 calculations
Pixel writing:     50 polygons × ~2,000 pixels = ~100,000 pixels/frame
                   At 30 FPS = ~3,000,000 pixels/second

Bottleneck: Pixel writing (inner loop)
Solution:   Assembly-optimized inner loops (68K/PowerPC)
            C fallbacks available in low_level_textures.c
```

**For Porting:**
- Replace `world_pixels` with your framebuffer
- Convert 8-bit palette output to 32-bit ARGB
- Use C fallbacks for assembly routines
- See Section 32 (Life of a Frame) for complete frame lifecycle

### Key Constants

```c
// Render tree limits
MAXIMUM_NODES = 512                  // Portal tree nodes
MAXIMUM_SORTED_NODES = 128           // Rendered polygons per frame
MAXIMUM_RENDER_OBJECTS = 72          // Sprites per frame

// Clipping limits
MAXIMUM_ENDPOINT_CLIPS = 64          // Vertex clip data entries
MAXIMUM_LINE_CLIPS = 256             // Edge clip data entries
MAXIMUM_CLIPPING_WINDOWS = 256       // Portal clip regions

// View settings
NORMAL_FIELD_OF_VIEW = 80           // Degrees horizontal
EXTRAVISION_FIELD_OF_VIEW = 130     // With powerup

// Distance thresholds
MINIMUM_OBJECT_DISTANCE = WORLD_ONE/20  // ~51 units

// Texture mapping
MAXIMUM_SCRATCH_TABLE_ENTRIES = 1024    // Edge table size
MAXIMUM_PRECALCULATION_TABLE_ENTRY_SIZE = 34  // Bytes per scanline/column
```

### Key Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `render.c` | 3,879 | Visibility, projection, render tree |
| `scottish_textures.c` | 1,200+ | Texture mapping routines |
| `screen.c` | ~800 | Framebuffer management |
| `shapes.c` | ~600 | Sprite/texture loading |
| `world.c` | ~400 | Coordinate transforms, trig tables |

---

*Next: [Chapter 6: Physics and Collision](06_physics.md) - How Marathon handles movement and collision detection*
