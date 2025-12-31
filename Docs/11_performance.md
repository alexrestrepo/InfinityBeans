# Chapter 11: Performance and Optimization

## Inner Loops and Optimization Strategies

> **For Porting:** Modern CPUs are vastly faster than 1995 hardware. Most optimizations are unnecessary, but understanding them helps with debugging. The C fallbacks for all assembly exist and work fine. Pre-compute shading tables at load time for best results.
>
> **Terminology:** Code examples use `screen` and `framebuffer` for the render target. See **[Appendix A: Glossary → Graphics Buffer Terminology](appendix_a_glossary.md#graphics-buffer-terminology)** for clarification.

---

## 11.1 What Problem Are We Solving?

Marathon needed to run on 1995 Macintosh hardware—68040 processors running at 25-40 MHz with no floating-point acceleration. Every cycle counted:

- **Rendering** - Drawing thousands of textured pixels per frame
- **Physics** - Collision detection against many polygons
- **AI** - Pathfinding for dozens of monsters

**The constraints:**
- No floating-point unit (68K Macs)
- 8-16 MB RAM typical
- 640×480 display at 15-30 FPS target

**Marathon's solution: Aggressive Optimization**

Marathon uses several key strategies to achieve playable performance on limited hardware. Understanding these helps with debugging and optimization on modern platforms.

---

## 11.2 Portal Culling: The Biggest Win

The single most important optimization is rendering only what's visible.

### Visibility Statistics

```
Average scene analysis:

Total polygons in level:    500-1000
Polygons after culling:     50-100
Reduction:                  10× fewer polygons to render!

This single optimization makes the game playable.
```

### How It Works

```
Player's view through portals:

     ┌──────────────────────────────────────────────────┐
     │  Polygon A (player is here)                      │
     │                                                  │
     │      Portal to B ──────►  ┌───────────────────┐  │
     │      (window)             │  Polygon B        │  │
     │                           │                    │  │
     │                           │   Portal to C ───►│  │
     │                           │   (smaller)       │  │
     │                           └───────────────────┘  │
     │                                                  │
     │                                                  │
     └──────────────────────────────────────────────────┘

Only polygons visible through portal chain are rendered.
Each portal clips the view narrower.
Polygons behind walls are never processed.
```

### Impact

Without portal culling, rendering would require processing ALL polygons:
- 1000 polygons × texture mapping = impossible on 68040

With portal culling:
- 50-100 polygons × texture mapping = 15-30 FPS achievable

---

## 11.3 Fixed-Point Mathematics

All calculations use integer math with implicit fractional bits.

### Why Not Floating-Point?

```
1995 Hardware Comparison:

68040 Integer ADD:     1 cycle
68040 Float ADD:       ~10 cycles (no FPU)
68040 Float MUL:       ~20 cycles (emulated in software!)

Integer is 10-20× faster on target hardware.
```

### The 16.16 Format

```
32-bit fixed-point number:

┌─────────────────────────────────┬─────────────────────────────────┐
│   16 bits: Integer part         │   16 bits: Fractional part      │
└─────────────────────────────────┴─────────────────────────────────┘

FIXED_ONE = 65536 = 1.0

Examples:
    0x00010000 = 1.0
    0x00008000 = 0.5
    0x00020000 = 2.0
    0x00018000 = 1.5

Precision: 1/65536 ≈ 0.000015 (plenty for game physics)
```

### Fixed-Point Operations

```c
// Addition - just add (fractions align)
fixed a = FIXED_ONE;      // 1.0
fixed b = FIXED_ONE / 2;  // 0.5
fixed c = a + b;          // 1.5 ✓

// Multiplication - multiply then shift
fixed result = (a * b) >> 16;  // Lose 16 bits of precision

// Division - shift then divide
fixed result = (a << 16) / b;  // Gain 16 bits first

// For portability, use macros
#define FIXED_MUL(a, b) (((a) * (b)) >> 16)
#define FIXED_DIV(a, b) (((a) << 16) / (b))
```

---

## 11.4 Pre-Computed Trigonometry

Sine and cosine lookups avoid expensive calculations.

### Trig Tables

```c
#define NUMBER_OF_ANGLES 512
fixed sine_table[NUMBER_OF_ANGLES];
fixed cosine_table[NUMBER_OF_ANGLES];
```

### Initialization

```c
void build_trig_tables(void) {
    for (int i = 0; i < NUMBER_OF_ANGLES; i++) {
        double angle = (i * 2 * PI) / NUMBER_OF_ANGLES;
        sine_table[i] = (fixed)(sin(angle) * TRIG_MAGNITUDE);
        cosine_table[i] = (fixed)(cos(angle) * TRIG_MAGNITUDE);
    }
}
```

### Usage

```c
// Look up instead of calculate
angle = player->facing;
fixed cos_facing = cosine_table[ANGLE_TO_INDEX(angle)];
fixed sin_facing = sine_table[ANGLE_TO_INDEX(angle)];

// Convert angle to table index
#define ANGULAR_BITS 9  // 512 angles
#define ANGLE_TO_INDEX(a) ((a) & (NUMBER_OF_ANGLES - 1))
```

---

## 11.5 Pre-Computed Shading Tables

Lighting calculations happen at load time, not render time.

### Shading Table Structure

```
Building shading tables:

For each brightness level (0-31 for 8-bit, 0-63 for 16/32-bit):
    For each palette color (0-255):
        Calculate: scaled_color = color × brightness / max_brightness
        Store in table[brightness][color]

                Brightness Level
                0        16        31
Color Index   (Dark)   (Mid)   (Bright)
    0         [  0  ] [  0  ] [  0  ]    (Black stays black)
    1         [ 12  ] [ 64  ] [ 128 ]
    2         [ 20  ] [ 80  ] [ 160 ]
    ...
    255       [ 127 ] [ 192 ] [ 255 ]    (White scales)
```

### Memory Usage

```
8-bit mode:   32 tables × 256 entries × 1 byte  =  8 KB
16-bit mode:  64 tables × 256 entries × 2 bytes = 32 KB
32-bit mode:  64 tables × 256 entries × 4 bytes = 64 KB

Small price for eliminating per-pixel lighting calculations!
```

### Usage During Rendering

```c
// Without shading tables (slow):
for (each pixel) {
    color = texture[u, v];
    r = (color.r * light_level) / 256;  // Multiply + divide
    g = (color.g * light_level) / 256;
    b = (color.b * light_level) / 256;
    framebuffer[x, y] = pack(r, g, b);
}

// With shading tables (fast):
for (each pixel) {
    palette_index = texture[u, v];
    framebuffer[x, y] = shading_table[light_level][palette_index];  // One lookup!
}
```

---

## 11.6 Assembly Optimizations

Critical inner loops had hand-tuned assembly for 68K and PowerPC.

### Complete Texture Mapper Function List

All functions exist in three bit-depth variants (scottish_textures.c:198-261):

**Horizontal (Floor/Ceiling) Mappers:**
```c
// Opaque texture mapping
void _texture_horizontal_polygon_lines8(struct bitmap_definition *texture,
    struct bitmap_definition *screen, struct view_data *view,
    struct _horizontal_polygon_line_data *data,
    short y0, short *x0_table, short *x1_table, short line_count);
void _texture_horizontal_polygon_lines16(...);  // 16-bit version
void _texture_horizontal_polygon_lines32(...);  // 32-bit version

// Landscape (sky) mapping
void _landscape_horizontal_polygon_lines8(...);
void _landscape_horizontal_polygon_lines16(...);
void _landscape_horizontal_polygon_lines32(...);
```

**Vertical (Wall) Mappers:**
```c
// Opaque texture mapping
void _texture_vertical_polygon_lines8(struct bitmap_definition *screen,
    struct view_data *view, struct _vertical_polygon_data *data,
    short *y0_table, short *y1_table);
void _texture_vertical_polygon_lines16(...);
void _texture_vertical_polygon_lines32(...);

// Transparent (sprite) mapping
void _transparent_texture_vertical_polygon_lines8(...);
void _transparent_texture_vertical_polygon_lines16(...);
void _transparent_texture_vertical_polygon_lines32(...);

// Tint mapping (colored overlays)
void _tint_vertical_polygon_lines8(..., word transfer_data);
void _tint_vertical_polygon_lines16(..., word transfer_data);
void _tint_vertical_polygon_lines32(..., word transfer_data);

// Static effect (TV noise)
void _randomize_vertical_polygon_lines8(..., word transfer_data);
void _randomize_vertical_polygon_lines16(..., word transfer_data);
void _randomize_vertical_polygon_lines32(..., word transfer_data);
```

### The Assembly/C Pattern

Marathon uses conditional compilation to select assembly or C implementations:

```c
// In scottish_textures.c header area
#ifdef env68k
    // 68K assembly versions declared as external
    #define EXTERNAL
    extern void _texture_vertical_polygon_lines8(...);
    // Assembly files: texture_68k.a
#endif

#ifdef envppc
    // PowerPC assembly versions
    extern void _texture_vertical_polygon_lines8(...);
    // Assembly files: texture_ppc.a
#endif

#ifndef EXTERNAL
    // C fallback - always present, used when no assembly available
    void _texture_vertical_polygon_lines8(
        struct bitmap_definition *screen,
        struct view_data *view,
        struct _vertical_polygon_data *data,
        short *y0_table, short *y1_table)
    {
        // Portable C implementation
        struct _vertical_polygon_line_data *line =
            (struct _vertical_polygon_line_data *)precalculation_table;

        for (short x = data->x0; x < data->x0 + data->width; x++)
        {
            pixel8 *shading_table = line->shading_table;
            pixel8 *texture = line->texture;
            pixel8 *dest = screen->row_addresses[0] + x;

            // Inner loop - this is what assembly optimizes
            long texture_y = line->texture_y;
            long texture_dy = line->texture_dy;

            for (short y = y0_table[x]; y < y1_table[x]; y++)
            {
                // Fetch texel, apply shading, write pixel
                pixel8 texel = texture[texture_y >> VERTICAL_TEXTURE_DOWNSHIFT];
                *dest = shading_table[texel];
                dest += screen->bytes_per_row;
                texture_y += texture_dy;
            }
            line++;
        }
    }
#endif
```

### Inner Loop Analysis

The texture mapper inner loop executes millions of times per frame:

```c
// Critical inner loop (executes ~200,000 times per frame at 640×480)
for (short y = y0; y < y1; y++)
{
    pixel8 texel = texture[texture_y >> 25];    // Texture fetch
    *dest = shading_table[texel];               // Shade + write
    dest += bytes_per_row;                       // Next row
    texture_y += texture_dy;                     // Advance texture
}

// 68K assembly optimizes this to ~8 cycles per pixel
// C version: ~15-20 cycles per pixel on 68040
```

**For porting**: The C fallbacks work perfectly. Modern CPUs execute this loop in under 1 cycle per pixel with out-of-order execution and cache prefetching.

---

## 11.7 Edge Table Caching

Bresenham's line algorithm is run once per polygon edge, results cached.

### The Problem

```
Drawing a textured polygon:

    P0 ──────────── P1
     ╲              │
      ╲             │
       ╲            │
        ╲           │
         P2 ────────┘

Need to fill 100 horizontal scanlines
Each scanline needs left/right edge X coordinates
Naive: Recalculate edge position each scanline = slow
```

### The Solution

```
Pre-compute edge tables:

left_edge_x[0] = P0.x               right_edge_x[0] = P1.x
left_edge_x[1] = P0.x + delta_x     right_edge_x[1] = P1.x
left_edge_x[2] = P0.x + 2*delta_x   right_edge_x[2] = P1.x - delta_x
...

Run Bresenham once per edge, store all X values
Then use the arrays for fast scanline filling
```

---

## 11.8 Staggered AI Updates

Monster AI is distributed across frames to prevent frame spikes.

### The Problem

```
Naive approach (all AI every frame):

Frame 1: 50 monsters × pathfinding = 50 expensive operations!
         Game runs at 5 FPS...

Frame 2: 50 monsters × pathfinding = 50 expensive operations!
         Still 5 FPS...
```

### The Solution

```
AI Load Distribution:

Frame N:    Monster 0 gets target search time
Frame N+1:  Monster 1 gets target search time
Frame N+2:  Monster 2 gets target search time
...

Frame N:    Monster 0 gets pathfinding time
Frame N+4:  Monster 1 gets pathfinding time  (1 per 4 frames)
Frame N+8:  Monster 2 gets pathfinding time
...

Result: Smooth 30 FPS maintained
```

### Implementation

```c
// Round-robin indices
dynamic_world->last_monster_index_to_get_time   // For targeting
dynamic_world->last_monster_index_to_build_path // For pathfinding

// Rules:
// - ONE monster gets expensive target search per frame
// - ONE monster gets pathfinding per 4 frames
// - Monsters without paths get immediate pathfinding
```

---

## 11.9 Memory Management

Classic Mac memory required careful handling.

### Mac Handles

```c
// Original Mac code:
Handle h = NewHandle(1024);   // Allocate relocatable memory
HLock(h);                     // Pin in place
byte* data = *h;              // Dereference
// ... use data
HUnlock(h);                   // Allow relocation
DisposeHandle(h);             // Free
```

### Modern Replacement

```c
// For porting: just use malloc
byte* data = malloc(1024);
// ... use data
free(data);

// No locking needed - modern OSes have virtual memory
```

---

## 11.10 See Also

- **[Chapter 5: Rendering System](05_rendering.md)** - Portal culling algorithm details
- **[Chapter 6: Physics and Collision](06_physics.md)** - Fixed-point math in physics
- **[Appendix D: Fixed-Point Conversion](appendix_d_fixedpoint.md)** - Complete fixed-point reference
- **[Chapter 8: Entity Systems](08_entities.md)** - AI load distribution details

---

## 11.11 Summary

Marathon's performance comes from smart algorithmic choices:

**Biggest Wins:**
- Portal culling: 10× polygon reduction
- Fixed-point math: 10-20× faster than float on 68K
- Pre-computed shading: Eliminates per-pixel lighting math

**Secondary Optimizations:**
- Trig lookup tables
- Edge caching for polygon filling
- Staggered AI updates

**For Modern Porting:**
- Use C implementations (plenty fast)
- Pre-convert shading tables to 32-bit at load time
- Don't worry about micro-optimizations

### Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `NUMBER_OF_ANGLES` | 512 | Trig table size |
| `FIXED_ONE` | 65536 | 1.0 in fixed-point |
| `TRIG_SHIFT` | 10 | Trig table precision |
| Shading tables (8-bit) | 32 × 256 | Light levels × colors |

### Key Source Files

| File | Purpose |
|------|---------|
| `scottish_textures.c` | Inner texture mapping loops |
| `world.c` | Trig tables, coordinate transforms |
| `render.c` | Portal culling |
| `monsters.c` | AI load distribution |

---

*Next: [Chapter 12: Data Structures Appendix](12_data_structures.md) - Complete type reference*
