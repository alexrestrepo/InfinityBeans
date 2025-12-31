# Appendix A: Glossary

## Definitions of Key Terms

---

## Core Concepts

### Action Flags
Bitmask representing player input state for a single game tick. Includes movement direction, firing state, and button presses. Queued at interrupt time, consumed by `update_players()`.

### Angle
Direction measurement in Marathon's 512-unit circle (0-511). 0=East, 128=North, 256=West, 384=South. Stored as `short`. See also: Binary angle.

### Binary Angle (Brad)
Marathon uses 9-bit angles (512 values per circle) for efficient lookup table indexing. Formula: `brad = degrees * 512 / 360`.

### Clipping Window
Rectangle defining visible area after portal culling. Each polygon in render tree has its own clip window, subtracting occluded regions.

### Collection
Group of related shapes (sprites/textures) loaded together. Examples: player collection, monster collection, wall texture collection. Max 32 collections.

### Control Panel
Interactive wall element (switches, pattern buffers, terminals). Processed in `update_control_panels()`.

### Data Fork
Standard file content portion of Mac files. All core Marathon data (Shapes, Sounds, Maps) uses data forks, readable with standard `fopen`/`fread`.

### Dynamic World
Runtime game state structure containing tick count, object counts, and game variables. Persists during level; saved to save files.

### Effect
Temporary visual object (explosion, spark, blood splash). Max 64 per map. Automatically removed when animation completes.

### Endpoint
Vertex in the map's 2D geometry. Shared by multiple lines. Contains x,y coordinates and supporting polygon index.

### Fixed-Point
16.16 format number representation. Upper 16 bits = integer, lower 16 bits = fraction. `FIXED_ONE = 0x10000 = 65536`. Avoids floating-point for deterministic simulation.

### Flood Map
Algorithm for pathfinding and area-of-effect calculations. Propagates values across polygon connections.

### High-Level Shape
Logical shape with facing angles. Resolves to low-level shape based on view angle.

### Line
Edge in map geometry connecting two endpoints. May be solid (wall) or transparent (portal).

### Low-Level Shape
Actual rendered image data. Consists of bitmap and animation frame information.

### Marathon WAD
Custom archive format for map files. **Not related to Doom WADs**. Contains tagged chunks for polygons, lights, objects, etc.

### Media
Liquid volume in a polygon (water, lava, goo, sewage, jjaro). Height controlled by linked light source intensity.

### Monster
AI-controlled enemy entity. 47 types across all environments. Uses behavior state machine for actions.

### Object
Any entity in the game world (monster, player, item, effect, scenery). Stored in global array, referenced by index.

### Permutation
Type-specific variation index. For monsters: sub-species. For items: specific item type. For sounds: random variant.

### Platform
Moving floor/ceiling (elevator, door, crushing ceiling). Controlled by activation and deactivation states.

### Polygon
Convex region of floor space. Contains height info, textures, lights, and references to adjacent polygons via lines.

### Portal
Transparent line connecting two polygons. Used for visibility determination in render tree.

### Projectile
Flying object with physics (bullet, rocket, grenade). Moves each tick, checks collisions, spawns effects on impact.

### Render Tree
Hierarchical structure of visible polygons built by portal traversal algorithm. Root is viewer's polygon.

### Resource Fork
Mac-specific structured data portion of files. **Only used for Images file** in Marathon. Not needed for core porting.

### Scenery
Static or animated decoration object. Optional solidity. Can be destructible. Max 20 animated scenery per map.

### Shading Table
Pre-computed lookup table mapping source color to shaded color at specific light intensity. One table per light level (32 levels typical).

### Shape
Visual image used for sprites and textures. Identified by shape descriptor (collection + shape index).

### Shape Descriptor
Packed identifier for a shape. `BUILD_DESCRIPTOR(collection, shape_index)` combines both into single `short`.

### Side
Wall surface data for a line. Contains texture references, transfer mode, and light source.

### Slot
Array position for objects, effects, projectiles, etc. "Slot used" flag indicates active entry.

### Static World
Level definition data that doesn't change during play. Environment code, mission flags, level name.

### Terminal
Computer interface for story content and level transitions. Uses markup language compiled to binary format.

### Tick
Single game simulation step. Target rate: 30 ticks/second (33.33ms per tick). All game timing based on ticks.

### Transfer Mode
Special rendering effect for textures. Examples: normal, static (noise), landscape (cylindrical), tinted.

### Trigger
Polygon that activates something when entered. Types: monster trigger, platform trigger, light trigger.

### World Unit
Base measurement unit. `WORLD_ONE = 1024`. Player height ≈ 819 units. 1 meter ≈ 1024 units.

---

## Map Geometry

| Term | Definition |
|------|------------|
| **Adjacent Polygon** | Polygon sharing a transparent line with another |
| **Ceiling Height** | Upper bound of polygon volume |
| **Floor Height** | Lower bound of polygon volume |
| **Line Flags** | Properties like solid, transparent, landscape |
| **Side Flags** | Rendering hints like full/split |

---

## Physics

| Term | Definition |
|------|------------|
| **Action Queue** | Buffer of pending player inputs |
| **Bounce** | Reflection of velocity on collision |
| **External Velocity** | Movement from external forces (explosions, currents) |
| **Gravity** | Downward acceleration (variable per environment) |
| **Terminal Velocity** | Maximum falling speed |

---

## Rendering

### Graphics Buffer Terminology

Marathon's source code uses several terms for pixel buffers that can be confusing. Here's the definitive reference:

| Term | What It Is | Context |
|------|------------|---------|
| **framebuffer** | Modern term for the pixel array displayed on screen | Used in porting documentation; not in original source |
| **world_pixels** | Marathon's offscreen render buffer (Mac GWorld) | Original variable name in `screen.c`; render target |
| **screen** | Parameter name in texture functions | `struct bitmap_definition *screen` = destination buffer |
| **destination** | Generic parameter name for output buffer | Same as `screen`, used in some functions |
| **bitmap_definition** | Struct wrapping any pixel buffer | Contains pixels, dimensions, row addresses |

**Relationship Diagram:**
```
┌─────────────────────────────────────────────────────────────────────┐
│                    GRAPHICS BUFFER HIERARCHY                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  bitmap_definition (struct)                                          │
│  ├── width, height         : Dimensions in pixels                   │
│  ├── bytes_per_row         : Stride (may include padding)           │
│  ├── row_addresses[]       : Pre-computed scanline pointers         │
│  └── pixels                : Raw pixel data                         │
│                                                                      │
│  world_pixels (Marathon's name)                                      │
│  = framebuffer (modern porting term)                                 │
│  = screen parameter in texture functions                             │
│  = destination parameter in render functions                         │
│                                                                      │
│  All refer to the same concept: the buffer render_view() draws to   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**For Porting:**
```c
// Marathon's world_pixels → your framebuffer
struct bitmap_definition render_buffer;
render_buffer.width = 640;
render_buffer.height = 480;
render_buffer.bytes_per_row = 640 * sizeof(uint32_t);
render_buffer.row_addresses[0] = (pixel8 *)your_framebuffer_pointer;
precalculate_bitmap_row_addresses(&render_buffer);

// Pass to render_view() as "destination" or "screen"
render_view(&view, &render_buffer);
```

### Other Rendering Terms

| Term | Definition |
|------|------------|
| **Affine Mapping** | Fast texture mapping without perspective correction (floors/ceilings) |
| **Bitmap** | Raw pixel data for shapes |
| **Column-Major** | Vertical strips stored contiguously (wall textures) |
| **FOV** | Field of View angle |
| **Perspective-Correct** | Texture mapping accounting for depth (walls) |
| **Row-Major** | Horizontal strips stored contiguously (floor/ceiling textures) |
| **Screen-Space** | 2D coordinates after projection |
| **World-Space** | 3D coordinates in game world |

---

## Networking

| Term | Definition |
|------|------------|
| **Action Flags** | Packed input state sent between players |
| **Deterministic Simulation** | All clients compute same result from same inputs |
| **Net Time** | Synchronized tick counter across network |
| **Ring Protocol** | Network topology where each node connects to next |

---

## Audio

| Term | Definition |
|------|------------|
| **Ambient Sound** | Continuous background audio for environment |
| **Local Sound** | 2D sound not positioned in world |
| **Sound Behavior** | How sound responds to distance and direction |
| **World Sound** | 3D positioned audio with attenuation |

---

## File Formats

| Term | Definition |
|------|------------|
| **Big-Endian** | Most significant byte first (Mac native order) |
| **Entry** | Named chunk in WAD file |
| **Tag** | 4-character code identifying data type ('POLY', 'LITE', etc.) |

---

## Mac-Specific

| Term | Definition |
|------|------------|
| **FSSpec** | File System Specification structure |
| **GrafPort** | QuickDraw graphics context |
| **GWorld** | Offscreen graphics buffer |
| **Handle** | Relocatable memory pointer (type**) |
| **PixMap** | Pixel map structure |
| **QuickDraw** | Mac 2D graphics API |
| **Sound Manager** | Mac audio API |
| **TickCount()** | Mac OS timing function (60 ticks/second) |

---

*Next: [Appendix B: Quick Reference](appendix_b_reference.md) - Common values and formulas*
