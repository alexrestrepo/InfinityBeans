# Marathon 2 → Full Beans Port: Complete Strategy

## Quick Start

| What | Where |
|------|-------|
| **Tutorial-style learning** | `chapters/` directory (32 chapters + 4 appendices, Crafting Interpreters style) |
| **Frame lifecycle** | `chapters/32_frame.md` - Essential for understanding the complete frame pipeline |
| **Quick reference** | `chapters/appendix_b_reference.md` - Constants, limits, formulas |
| **Porting context** | CLAUDE.md (memory handles, resource forks, decisions made) |
| **This document** | Step-by-step checklist with chapter references |

**First steps**: Read `chapters/32_frame.md` (Life of a Frame) → then start with Milestone 1 below.
**For rendering**: See `chapters/05_rendering.md` for portal visibility and texture mapping.
**For file formats**: See `chapters/10_file_formats.md` and `chapters/31_resource_forks.md`.

---

## Overview

**Good news**: Marathon 2's rendering engine is almost entirely platform-independent! The core 3D renderer (render.c - 3,879 lines) is pure C software rendering that just needs a framebuffer. Only ~11 Mac-specific files need replacement.

> **📚 Technical Reference**: Before starting, read the tutorial chapters for comprehensive understanding:
> - `chapters/04_world.md` - Polygon connectivity and data structures
> - `chapters/05_rendering.md` - Portal culling and texture mapping pipelines
> - `chapters/06_physics.md` - Fixed-point math and collision algorithms
> - `chapters/10_file_formats.md` - WAD and shape file parsing details
> - **`chapters/32_frame.md`** - Complete frame pipeline walkthrough (essential for porting!)
> - `chapters/appendix_b_reference.md` - All constants, limits, and formulas at a glance

### Chapter Reference by Milestone

| Milestone | Primary Chapters | Key Implementation Details |
|-----------|------------------|---------------------------|
| M1: Setup | Ch 1-2 (Intro, Source Org) | Module organization, build targets |
| M2: Platform Layer | Ch 24 (cseries.lib), App D (Fixed-Point) | `uint16_t`/`int32_t` types, swap_16/32 |
| M3: File Loading | Ch 10 (File Formats), Ch 31 (Resource Forks) | WAD header, data fork files |
| M4: Shapes/Textures | Ch 10, Ch 19 (Shape Animation) | 32 collection headers, bitmap_definition |
| M5: Input | Ch 7 (Game Loop) | 32-bit action_flags bitmask |
| M6: Graphics Bridge | Ch 5 (Rendering), Ch 24 (Pixel Types) | world_pixels buffer, 640×480 default |
| M7: First Frame | Ch 4 (World), Ch 5 (Rendering), Ch 32 (Frame) | render_view(), 31 shading levels |
| M8: Game Loop | Ch 7 (Game Loop), Ch 6 (Physics), Ch 25 (Media) | 33ms tick (30 Hz), update_world() |
| M9: Combat | Ch 8 (Entities), Ch 16 (Damage) | 47 monster types, damage system |
| M10: Audio | Ch 13 (Sound) | 5 sound channels, 3D positioning |
| M11: HUD | Ch 21 (HUD), Ch 20 (Automap), Ch 28 (Terminal) | radar sweep, line primitives |
| M12: Optimization | Ch 11 (Performance), App D (Fixed-Point) | inner loop optimizations |

> 💡 **Essential Reading**: `chapters/32_frame.md` provides a complete walkthrough of one frame from input to display - invaluable for understanding how all systems connect.

**Target**: Full Beans provides perfect infrastructure:
- 32-bit ARGB framebuffer (direct pixel access)
- Cross-platform window/input (macOS/Windows/Linux)
- Simple event loop
- Minimal dependencies

**Challenge areas**:
- Audio (need to add library like miniaudio)
- Mac assembly optimizations (need C fallbacks)
- 8-bit palette mode → 32-bit conversion (see `chapters/appendix_b_reference.md` for color conversion)

---

## PHASE 1: Foundation Layer (Milestone 1-2)
**Goal**: Get a window open and establish build system

### Milestone 1: Project Setup & Build System
**Estimated effort**: 2-4 hours

> 📚 **Reference**: [Ch 1-2](chapters/01_introduction.md) for architecture, [Ch 3](chapters/03_engine_overview.md) for module organization

**Tasks**:
- [ ] Create new directory structure:
  ```
  marathon-port/
  ├── src/
  │   ├── platform/        # Full Beans + platform abstraction
  │   ├── marathon/        # Marathon 2 source (copied)
  │   └── main.c           # New entry point
  ├── data/                # WAD files
  └── Makefile
  ```
- [ ] Copy Marathon 2 core files (~60 C files):
  - Exclude: `*_macintosh.c`, `network_*.c`, assembly files
  - Include: render.c, map.c, physics.c, monsters.c, weapons.c, etc.
- [ ] Copy cseries.lib/ utility files (entire directory)
- [ ] Integrate fenster.h into build
- [ ] Create minimal main.c that opens 640×480 window
- [ ] Set up compiler flags:
  - Remove Mac-specific: `-framework Carbon`, MPW pragmas
  - Add: `-I` paths, `-std=c99` or `-std=gnu99`
  - Define: `-DSDL_PORT` or `-DFENSTER_PORT` for conditional compilation

**Test**:
- [ ] Compiles without errors
- [ ] Opens 640×480 window with title "Marathon 2"
- [ ] Window responds to close button
- [ ] Framebuffer clears to black

**Files to create**:
- `src/main.c` - Entry point with Fenster initialization
- `Makefile` - Build system
- `src/platform/platform.h` - Platform abstraction header

---

### Milestone 2: Platform Abstraction Layer
**Estimated effort**: 4-8 hours

**Goal**: Create clean abstractions for Mac-specific functions

> 📚 **Reference**: [Ch 24](chapters/24_cseries.md) for types/macros, [App D](chapters/appendix_d_fixedpoint.md) for fixed-point math

**Tasks**:
- [ ] **Create `platform/memory.c`** - Replace Mac Memory Manager:
  ```c
  void* platform_allocate(size_t size);  // Simple malloc wrapper
  void platform_free(void* ptr);         // Simple free wrapper
  ```

  **Note on handles**: Marathon uses `type**` (handles) for shape collections and shading tables. These were needed on classic Mac OS for relocatable memory (to avoid heap fragmentation). Modern systems don't need this - replace with regular pointers:
  - Change `struct collection_definition **collection` → `*collection` in structs
  - Replace `NewHandle(size)` → `malloc(size)`
  - Replace `HLock(h)` / `HUnlock(h)` → remove (not needed)
  - Replace `DisposeHandle(h)` → `free(h)`
  - Remove dereference: `*handle` → `handle`
- [ ] **Create `platform/files.c`** - Replace File Manager:
  ```c
  typedef struct {
    FILE* fp;
    char path[256];
  } platform_file_handle;

  bool platform_open_file(const char* name, platform_file_handle* out);
  bool platform_read_file(platform_file_handle* f, void* buffer, size_t* bytes);
  bool platform_close_file(platform_file_handle* f);
  bool platform_find_file(const char* name, char* path_out);
  ```
- [ ] **Create `platform/timing.c`** - Replace TickCount():
  ```c
  uint32_t platform_get_ticks(void);  // Milliseconds
  void platform_sleep(uint32_t ms);
  ```
- [ ] **Create `platform/graphics.c`** - Framebuffer abstraction:
  ```c
  typedef struct {
    uint32_t* pixels;
    int width, height, pitch;
  } platform_framebuffer;

  void platform_present_framebuffer(struct fenster* f);
  ```
- [ ] **Stub out `platform/sound.c`** (implement later):
  ```c
  void platform_init_sound(void) { /* stub */ }
  void platform_play_sound(int id, float volume) { /* stub */ }
  ```

**Replace in Marathon source**:
- [ ] Find all `NewPtr()`, `DisposePtr()` → `malloc()` / `free()`
- [ ] Find all `NewHandle()`, `DisposeHandle()` → `malloc()` / `free()`
- [ ] Find all `HLock()`, `HUnlock()`, `MoveHHi()` → remove (no-ops on modern systems)
- [ ] Find handle types `type**` → change to `type*` (remove double indirection)
- [ ] Find all `*handle` dereferences → change to `handle`
- [ ] Find all `TickCount()` → platform_get_ticks()
- [ ] Add `#ifndef FENSTER_PORT` around Mac includes

**Test**:
- [ ] All Marathon files compile with platform layer
- [ ] No Mac headers included (except in #ifdef blocks)
- [ ] Memory allocation/deallocation works
- [ ] Timing functions return reasonable values

---

## PHASE 2: Data Loading (Milestone 3-4)
**Goal**: Load Marathon data files (maps, textures, sounds)

**File Format Summary** (see "Critical Clarification" section below for details):
- **Marathon WAD** (custom format, NOT Doom) - maps only, readable with fopen/fread
- **Binary data files** (Shapes/Sounds) - readable on all platforms with standard file I/O!
- **Resource forks** - ONLY for Images file - NOT needed for basic port

### Milestone 3: File Loading System
**Estimated effort**: 8-12 hours

> 📚 **Reference**: [Ch 10](chapters/10_file_formats.md) for WAD structure and shape file format, [Ch 31](chapters/31_resource_forks.md) for resource fork clarification

**Tasks**:
- [ ] **Port `wad.c`** - Marathon's custom map file format (mostly portable):
  - [ ] Replace `FSSpec` with `platform_file_handle`
  - [ ] Replace `BlockMove()` with `memcpy()`
  - [ ] Replace Mac file I/O with stdio
  - [ ] Keep checksumming, tag system intact
- [ ] **Port `game_wad.c`** - Marathon-specific map handling
- [ ] **Create `platform/file_reader.c`** - Simple binary file reader:
  - [ ] Open files with standard fopen()
  - [ ] Byte swapping utilities (big-endian → little-endian)
  - [ ] Read shape collection headers
  - [ ] Read map WAD structures
- [ ] **Create `platform/file_finder.c`**:
  - [ ] Search for map files and data files
  - [ ] Check `./data/` directory
  - [ ] Check executable directory
  - [ ] Environment variable `MARATHON_DATA_PATH`

**Test with real Marathon data**:
- [ ] Copy Marathon 2 data files to `data/`:
  - Map file (e.g., `Marathon 2`) - Marathon WAD format
  - `Shapes16` - Binary data fork file
  - `Sounds16` - Binary data fork file
- [ ] Load map file and verify checksums
- [ ] Print map WAD directory entries
- [ ] Extract a tag from map WAD and verify contents
- [ ] Read collection header table from Shapes file
- [ ] Verify byte swapping works correctly (big-endian → native)

**Note on Images file**: The Images file uses resource forks for interface graphics (PICT resources). For initial porting, you can:
- Skip it entirely (stub out with colored rectangles)
- Extract once on macOS and convert to PNG
- Use Aleph One's pre-converted graphics
See CLAUDE.md for detailed resource fork extraction instructions.

---

### Milestone 4: Shape/Texture Loading
**Estimated effort**: 10-14 hours

> 📚 **Reference**: [Ch 19](chapters/19_shapes.md) for collections/bitmaps, [Ch 24](chapters/24_cseries.md) for pixel types, [Ch 10](chapters/10_file_formats.md) for binary layout

**Tasks**:
- [ ] **Port `shapes.c`** - Shape collection management:
  - [ ] Remove `#include <shapes_macintosh.c>` conditional
  - [ ] Keep collection_definition structures
- [ ] **Create `platform/shapes.c`** to replace `shapes_macintosh.c`:
  - [ ] Load shape collections from **binary data file** (not resource fork!)
  - [ ] Read collection header table (32 entries at file start)
  - [ ] Parse collection_definition structures
  - [ ] Handle byte swapping (all data is big-endian)
  - [ ] **Color conversion**: 8-bit palette → 32-bit ARGB
    ```c
    // Marathon uses 16-bit color tables (RGB555)
    struct rgb_color { uint16 red, green, blue; };

    // Convert to 32-bit ARGB
    uint32_t convert_color(struct rgb_color* c) {
      uint8_t r = c->red >> 8;
      uint8_t g = c->green >> 8;
      uint8_t b = c->blue >> 8;
      return 0xFF000000 | (r << 16) | (g << 8) | b;
    }
    ```
  - [ ] Build shading tables for 32-bit mode
  - [ ] Convert bitmap data format
- [ ] **Port texture loading**:
  - [ ] Load texture collections
  - [ ] Decompress RLE-encoded bitmaps
  - [ ] Build texture->bitmap mappings

**Test**:
- [ ] Load shape collection successfully
- [ ] Print texture dimensions and counts
- [ ] Render a single texture to framebuffer (test blit)
- [ ] Verify shading tables have correct gradient

---

## PHASE 3: Minimal Rendering (Milestone 5-7)
**Goal**: Render the first 3D frame

### Milestone 5: Input System
**Estimated effort**: 4-6 hours

> 📚 **Reference**: [Ch 7](chapters/07_game_loop.md) for action flags and input integration

**Tasks**:
- [ ] **Create `platform/input.c`** to replace `vbl_macintosh.c`:
  ```c
  // Input state structure
  typedef struct {
    uint32_t action_flags;  // Marathon's action bitmask
    int16_t yaw_delta;
    int16_t pitch_delta;
    int16_t velocity_delta;
  } platform_input_state;

  void platform_poll_input(struct fenster* f, platform_input_state* out);
  ```
- [ ] **Map Fenster keys to Marathon actions**:
  - [ ] Arrow keys → turn/move (or WASD)
  - [ ] Space → primary trigger
  - [ ] Ctrl/Cmd → secondary trigger
  - [ ] Tab → action key (switches, terminals)
  - [ ] 1-9 → weapon selection
  - [ ] M → map toggle
- [ ] **Implement in parse_keymap() replacement**:
  - [ ] Convert fenster->keys[] to Marathon action_flags
  - [ ] Handle key auto-repeat properly
  - [ ] Mouse: fenster->x, y → yaw/pitch deltas
  - [ ] Mouse button → fire

**Test**:
- [ ] Print action_flags each frame
- [ ] Verify key presses detected
- [ ] Verify mouse movement creates deltas
- [ ] Test all weapon select keys

---

### Milestone 6: Graphics Bridge
**Estimated effort**: 6-10 hours

> 📚 **Reference**: [Ch 5](chapters/05_rendering.md) for render flow, [Ch 24](chapters/24_cseries.md) for pixel types, [App A](chapters/appendix_a_glossary.md#graphics-buffer-terminology) for buffer terminology

**Tasks**:
- [ ] **Port `screen.c`** - Screen management:
  - [ ] Remove QuickDraw/GWorld code
  - [ ] Create `world_pixels` buffer (matches Marathon's format)
  - [ ] **Key decision**: Render at what depth?
    - **Option A** (simpler): Render directly to 32-bit ARGB
    - **Option B** (more authentic): Render to 16-bit, convert to 32-bit
  - [ ] Implement `render_screen()` replacement
- [ ] **Modify `scottish_textures.c`**:
  - [ ] **Remove assembly**: Comment out `.a`/`.s` includes
  - [ ] Use C fallback texture mappers (already exists for non-68K/PPC)
  - [ ] Modify for 32-bit pixel format:
    ```c
    // Old: writes byte palette index
    *pixel_ptr = shading_table[texture_pixel];

    // New: writes 32-bit color
    *pixel_ptr = color_table[shading_table[texture_pixel]];
    ```
- [ ] **Create framebuffer copy routine**:
  ```c
  void blit_to_fenster(uint32_t* marathon_buffer, struct fenster* f) {
    memcpy(f->buf, marathon_buffer, f->width * f->height * sizeof(uint32_t));
  }
  ```

**Test**:
- [ ] Allocate world_pixels buffer
- [ ] Clear to solid color, blit to window
- [ ] Try rendering a test pattern
- [ ] Verify no crashes in texture mapping code

---

### Milestone 7: Render First Frame
**Estimated effort**: 10-16 hours

**Goal**: See actual Marathon 3D rendering

> 📚 **Reference**: [Ch 4](chapters/04_world.md) for polygons, [Ch 5](chapters/05_rendering.md) for render_view(), [Ch 32](chapters/32_frame.md) for complete frame walkthrough

**Tasks**:
- [ ] **Port core rendering files** (mostly portable):
  - [ ] `render.c` - Main renderer (should compile as-is!)
  - [ ] `textures.c` - Texture management
  - [ ] `low_level_textures.c` - Texture utils
  - [ ] Verify no Mac dependencies
- [ ] **Port lighting system** (needed for proper shading):
  - [ ] `lightsource.c` - Dynamic lights and shading (portable)
  - [ ] Verify shading table generation works
- [ ] **Port map system**:
  - [ ] `map.c` - Level geometry (portable)
  - [ ] `map_constructors.c` - Map building
  - [ ] Replace Mac types in headers (Ptr → void*)
- [ ] **Port screen fades**:
  - [ ] `fades.c` - Screen fade effects (portable)
- [ ] **Initialize rendering subsystem**:
  ```c
  allocate_render_memory();
  allocate_texture_tables();
  initialize_view_data(&view);
  ```
- [ ] **Load a map**:
  ```c
  // Load map from WAD
  entry_point map_entry;
  map_entry = get_indexed_entry_point(...);
  import_level(...);
  ```
- [ ] **Render one frame**:
  ```c
  struct view_data view;
  view.origin = player_position;
  view.yaw = player_angle;
  view.pitch = 0;

  render_view(&view, destination_bitmap);
  ```

**Test**:
- [ ] Render system initializes without crash
- [ ] Map loads successfully
- [ ] Lighting system initializes (shading tables populated)
- [ ] First call to render_view() completes
- [ ] Window shows something (even if wrong)
- [ ] Dynamic lighting changes are visible
- [ ] No segfaults or null pointer crashes

**Debug checklist**:
- [ ] Are textures loaded?
- [ ] Are shading tables valid?
- [ ] Is player position valid?
- [ ] Are polygon counts reasonable?
- [ ] Does rendering tree build?

---

## PHASE 4: Playable Game (Milestone 8-10)
**Goal**: Basic playable single-player experience

### Milestone 8: Game Loop Integration
**Estimated effort**: 8-12 hours

> 📚 **Reference**: [Ch 7](chapters/07_game_loop.md) for update_world() and 30 Hz timing, [Ch 6](chapters/06_physics.md) for physics, [Ch 25](chapters/25_media.md) for water/lava

**Tasks**:
- [ ] **Port `marathon2.c`** - Main game logic:
  - [ ] Remove Mac initialization code
  - [ ] Keep game state machine
  - [ ] Keep world update loop (30 ticks/sec)
- [ ] **Port player system**:
  - [ ] `player.c` - Player state/control
  - [ ] Integrate platform_input with player control
- [ ] **Port physics**:
  - [ ] `physics.c` - Movement/collision (portable)
  - [ ] `world.c` - World queries (portable)
- [ ] **Port world systems** (needed before combat):
  - [ ] `platforms.c` - Elevators, doors (portable) - called by update_platforms()
  - [ ] `media.c` - Water, lava physics (portable) - affects player movement
  - [ ] `flood_map.c` - Zone calculations (portable) - needed for pathfinding
- [ ] **Initialize support systems**:
  - [ ] `allocate_pathfinding_memory()` - Called in initialize_marathon()
  - [ ] `allocate_flood_map_memory()` - Called in initialize_marathon()
- [ ] **Implement game loop**:
  ```c
  int main() {
    struct fenster f = {...};
    fenster_open(&f);

    initialize_marathon();
    start_game();

    uint32_t last_tick = platform_get_ticks();

    while (fenster_loop(&f) == 0) {
      uint32_t current_tick = platform_get_ticks();

      // Run at 30 ticks/sec
      while (current_tick - last_tick >= 33) {
        platform_input_state input;
        platform_poll_input(&f, &input);

        update_world_one_tick(input.action_flags);

        last_tick += 33;
      }

      render_screen(&f);
    }
  }
  ```

**Test**:
- [ ] Game loop runs at steady 30 ticks/sec
- [ ] Player moves forward/backward
- [ ] Player turns left/right
- [ ] Collision detection works
- [ ] View updates each frame
- [ ] Doors open/close when triggered
- [ ] Platforms/elevators move correctly
- [ ] Water/lava affects player (swimming, damage)
- [ ] No stuttering or slowdown

---

### Milestone 9: Combat & Entities
**Estimated effort**: 12-16 hours

> 📚 **Reference**: [Ch 8](chapters/08_entities.md) for monsters/AI, [Ch 16](chapters/16_damage.md) for damage system

**Tasks**:
- [ ] **Port entity systems**:
  - [ ] `monsters.c` - AI and monsters (mostly portable)
  - [ ] `projectiles.c` - Bullets, grenades (portable)
  - [ ] `weapons.c` - Weapon logic (portable)
  - [ ] `items.c` - Pickups (portable)
  - [ ] `effects.c` - Visual effects (portable)
  - [ ] `scenery.c` - Static objects (portable)
- [ ] **Port AI support**:
  - [ ] `pathfinding.c` - Monster AI navigation (portable)
  - [ ] Verify flood_map integration (from M8)

**Test**:
- [ ] Monsters spawn and are visible
- [ ] Monsters move toward player
- [ ] Player can shoot weapons
- [ ] Projectiles spawn and move
- [ ] Hit detection works (player damages monsters)
- [ ] Monster hit detection works (player takes damage)
- [ ] Item pickups work
- [ ] Visual effects (explosions, sparks) display correctly

---

### Milestone 10: Basic Audio
**Estimated effort**: 8-12 hours

**Strategy**: Add miniaudio (single-header) or SDL2_mixer

> 📚 **Reference**: [Ch 13](chapters/13_sound.md) for sound channels and 3D audio, [Ch 10](chapters/10_file_formats.md) for sound file format

**Tasks** (using miniaudio):
- [ ] Add miniaudio.h to project
- [ ] **Implement `platform/sound.c`**:
  ```c
  typedef struct {
    ma_engine engine;
    ma_sound sounds[MAX_SOUNDS];
  } platform_audio;

  bool platform_init_audio();
  void platform_play_sound(int sound_id, float volume, float pitch);
  void platform_stop_sound(int channel);
  ```
- [ ] **Port sound loading**:
  - [ ] Load sound definitions from WAD
  - [ ] Decode Marathon sound format
  - [ ] Convert to format miniaudio accepts
- [ ] **Port `game_sound.c`**:
  - [ ] Remove `#include <sound_macintosh.c>`
  - [ ] Replace Mac Sound Manager calls
  - [ ] Keep 3D positioning logic
  - [ ] Map to platform_sound API
- [ ] **Implement 3D audio**:
  - [ ] Calculate volume based on distance
  - [ ] Calculate stereo pan based on angle
  - [ ] Apply pitch variation

**Test**:
- [ ] Sound system initializes
- [ ] Weapon fire sounds play
- [ ] Monster sounds play
- [ ] Ambient sounds loop
- [ ] No audio glitches or crashes
- [ ] Volume changes with distance
- [ ] Stereo panning works

---

## PHASE 5: Polish & Optimization (Milestone 11-12)

### Milestone 11: HUD & Interface
**Estimated effort**: 6-10 hours

> 📚 **Reference**: [Ch 21](chapters/21_hud.md) for HUD/radar, [Ch 28](chapters/28_terminals.md) for terminals, [Ch 20](chapters/20_automap.md) for overhead map, [Ch 22](chapters/22_fades.md) for screen fades

**Tasks**:
- [ ] **Port `interface.c`** - HUD rendering:
  - [ ] Remove QuickDraw dependencies
  - [ ] Draw directly to framebuffer
  - [ ] Motion sensor
  - [ ] Health/oxygen bars
  - [ ] Ammo counter
  - [ ] Weapon sprite
- [ ] **Port `overhead_map.c`**:
  - [ ] Remove QuickDraw drawing
  - [ ] Draw using line primitives to framebuffer
- [ ] **Create simple text renderer**:
  - [ ] Use Full Beans' microui font, or
  - [ ] Load Marathon's interface fonts
  - [ ] Render score, messages

**Test**:
- [ ] HUD displays correctly
- [ ] Motion sensor shows monsters
- [ ] Weapon sprite changes on switch
- [ ] Map overlay (M key) works
- [ ] Player stats update in real-time

---

### Milestone 12: Performance Optimization
**Estimated effort**: 4-8 hours

> 📚 **Reference**: [Ch 11](chapters/11_performance.md) for optimization strategies, [App D](chapters/appendix_d_fixedpoint.md) for fixed-point math

**Tasks**:
- [ ] **Profile rendering**:
  - [ ] Measure frame time
  - [ ] Identify bottlenecks
- [ ] **Optimize texture mapping**:
  - [ ] Ensure tight loops
  - [ ] Consider SIMD if available
  - [ ] Cache shading table lookups
- [ ] **Optimize color conversion**:
  - [ ] Pre-convert all colors to 32-bit
  - [ ] Avoid per-pixel conversion
- [ ] **Resolution options**:
  - [ ] Support 640×480, 640×400, 800×600
  - [ ] Optional 2x upscaling for modern displays
- [ ] **Build optimized release**:
  - [ ] Enable `-O2` or `-O3`
  - [ ] Test performance difference

**Test**:
- [ ] Measure FPS in various scenarios
- [ ] Target: 30+ FPS at 640×480 on modern CPU
- [ ] Verify no visual glitches from optimizations
- [ ] Compare vs debug build performance

---

## SUCCESS CRITERIA (Final Checklist)

### Core Functionality
- [ ] Game launches without errors
- [ ] First level loads and renders
- [ ] Player can move, look, and navigate
- [ ] Textures render correctly
- [ ] Lighting looks appropriate
- [ ] Weapons fire and switch
- [ ] Monsters spawn, move, attack
- [ ] Combat works bidirectionally
- [ ] Sound effects play
- [ ] HUD displays game state
- [ ] Map overlay shows level

### Technical Quality
- [ ] No memory leaks (run valgrind)
- [ ] No segfaults or crashes
- [ ] Stable 30 ticks/sec game logic
- [ ] Acceptable frame rate (30+ FPS)
- [ ] Portable (compiles on macOS/Linux/Windows)
- [ ] Clean abstraction layer

### Polish
- [ ] Reasonable error messages
- [ ] Graceful handling of missing data files
- [ ] Command-line options (--data-path, --resolution)
- [ ] README with build instructions

---

## ESTIMATED TOTAL EFFORT

| Phase | Milestones | Hours |
|-------|-----------|-------|
| Phase 1: Foundation | 1-2 | 6-12 |
| Phase 2: Data Loading | 3-4 | 18-26 |
| Phase 3: Minimal Rendering | 5-7 | 20-32 |
| Phase 4: Playable Game | 8-10 | 28-40 |
| Phase 5: Polish | 11-12 | 10-18 |
| **TOTAL** | **12 milestones** | **82-128 hours** |

**Realistic timeline**: 2-3 weeks full-time, or 6-8 weeks part-time

---

## RECOMMENDED DEVELOPMENT ORDER

**Critical path** (do in sequence):
1. M1 → M2 → M3 → M4 → M5 (Foundation + data + input)
2. M6 → M7 (Graphics bridge + first render)
3. M8 → M9 (Game loop + entities)

**Can parallelize**:
- M10 (audio) can be done anytime after M3
- M11 (HUD) can be done anytime after M7
- M12 (optimization) is truly final

**Quick validation strategy**:
- After M7: Should see static 3D rendering
- After M8: Should be able to walk around
- After M9: Should be able to fight monsters

---

## TECHNICAL APPENDIX

### Marathon 2 Mac Dependencies Summary

**IMPORTANT: File Format Clarification**

Marathon 2 uses TWO different file formats:

1. **Marathon WAD format** (custom, NOT Doom WADs):
   - Used ONLY for map/level data
   - Platform-independent binary format
   - Tag-based structure (defined in `wad.c`, `wad.h`)
   - Files: "Marathon 2" (map file), saved games
   - Big-endian byte order

2. **Binary data files** (data fork format):
   - Used for shapes (textures/sprites) and sounds
   - Simple binary files readable with standard fopen/fread on ANY platform
   - Files: `Shapes8`, `Shapes16`, `Sounds8`, `Sounds16`
   - Collection header table at file offset 0
   - Big-endian byte order
   - **No Mac-specific APIs needed!**

3. **Resource forks** (ONLY for optional files):
   - Used ONLY for Images file (interface graphics - PICT resources)
   - Also used for optional scenario files
   - **NOT needed for basic gameplay!**
   - Can be stubbed out or extracted once and converted to PNG
   - See CLAUDE.md for resource fork extraction guide

**Mac-specific files to replace (11 files)**:
1. `game_window_macintosh.c` - Window rendering
2. `interface_macintosh.c` - Menus/dialogs
3. `sound_macintosh.c` - Sound Manager
4. `vbl_macintosh.c` - Input handling
5. `files_macintosh.c` - File I/O
6. `shapes_macintosh.c` - Shape loading
7. `wad_macintosh.c` - WAD searching
8. `overhead_map_macintosh.c` - Map drawing
9. `wad_prefs_macintosh.c` - Preferences
10. `preprocess_map_mac.c` - Map utilities
11. Network files (can stub for single-player)

**Platform-independent core (~60 files)**:
- render.c, map.c, physics.c, player.c
- monsters.c, projectiles.c, weapons.c, items.c
- platforms.c, lightsource.c, effects.c, media.c
- scottish_textures.c, textures.c, low_level_textures.c
- wad.c, game_wad.c, shapes.c
- marathon2.c (main game logic)

> 📚 **Reference**: See [Appendix C](chapters/appendix_c_files.md) for complete file-by-file reference with portability status.

### Full Beans / Fenster API Reference

**Window creation**:
```c
struct fenster f = {
  .title = "Marathon 2",
  .width = 640,
  .height = 480
};
uint32_t buffer[640 * 480];
f.buf = buffer;

fenster_open(&f);
while (fenster_loop(&f) == 0) {
  // Render frame
  // f.buf[y * 640 + x] = 0xAARRGGBB;
}
fenster_close(&f);
```

**Input**:
```c
// Keyboard: f.keys[key_code] (1 = pressed, 0 = released)
// Mouse: f.x, f.y (position), f.mouse (1 = button down)
// Time: fenster_time() returns milliseconds
```

**Framebuffer format**:
- 32-bit ARGB (0xAARRGGBB)
- Direct memory access
- Software rendering

### Color Conversion Notes

Marathon uses 16-bit color tables (RGB555 format):
```c
struct rgb_color {
  uint16 red;    // 0-65535 range
  uint16 green;
  uint16 blue;
};
```

Convert to 32-bit ARGB:
```c
uint32_t marathon_to_argb(struct rgb_color* c) {
  uint8_t r = c->red >> 8;    // Take high byte
  uint8_t g = c->green >> 8;
  uint8_t b = c->blue >> 8;
  return 0xFF000000 | (r << 16) | (g << 8) | b;
}
```

For shading tables, pre-convert entire palette to 32-bit to avoid per-pixel conversion overhead.

---

## Additional Resources

**Marathon source code locations**:
- Main directory: `./marathon2/`
- Utilities: `./cseries.lib/`

**Full Beans framework**:
- Location: (configure path to your Full Beans installation)
- Key files: `fenster.h`, `renderer.c/h`, `main.c` (example)

**Recommended tools**:
- Audio: miniaudio.h (single-header, cross-platform)
- Build: Make or CMake
- Debugging: gdb, lldb, valgrind (for memory leaks)

**This is an achievable port! The Marathon engine is surprisingly clean and most code is portable C. Good luck!**

---

## FILE FORMAT REFERENCE

For detailed technical specifications on how to read Marathon's file formats, see [Chapter 10: File Formats](chapters/10_file_formats.md) which contains:

### Marathon WAD Format (Maps)
- Complete binary structure documentation
- WAD header, directory, and entry header layouts
- Tag-based data organization
- Step-by-step reading instructions with code examples
- Endianness notes (big-endian/Mac byte order)

### Shape File Format (Textures/Sprites)
- Collection header table structure
- How to read from data fork (works on all platforms!)
- Collection definition parsing
- High-level and low-level shape structures
- Bitmap formats (raw and RLE compressed)
- Color table format

### Key Insights from File Format Analysis:

**Good news for porting**:
1. **Shapes and sounds are NOT in resource forks!** They're regular data fork binary files
2. You can read Shapes16/Sounds16 files on any platform with standard `fopen`/`fread`
3. Only map files use the WAD format, which is straightforward to parse
4. Resource forks are ONLY used for Images file (not essential) and optional scenarios
5. All formats use big-endian byte order (may need byte swapping on x86)

**File reading strategy**:
```c
// Reading shapes is simple:
FILE* fp = fopen("Shapes16", "rb");
struct collection_header headers[32];
fread(&headers, sizeof(struct collection_header), 32, fp);

// Then seek to collection offsets and read data
fseek(fp, headers[collection_id].offset16, SEEK_SET);
// Parse collection_definition structure...
```

See [Chapter 10](chapters/10_file_formats.md) for complete code examples and structure definitions.

---

## CRITICAL CLARIFICATION: What Uses Resource Forks?

**This is important to understand before starting your port:**

### Files You CAN Read Directly (No Resource Fork Issues):
✓ **Shapes16** - Data fork binary file (fopen/fread works!)
✓ **Shapes8** - Data fork binary file
✓ **Sounds16** - Data fork binary file
✓ **Sounds8** - Data fork binary file
✓ **Map files** - Marathon WAD format (fopen/fread works!)
✓ **Saved games** - Marathon WAD format

### Files That Use Resource Forks (Optional for Basic Port):
✗ **Images** - PICT resources for interface graphics (can stub or pre-extract)
✗ **Scenario files** - Optional custom campaign assets (can skip)
✗ **Music** - QuickTime data (can skip)

### The Extraction Confusion

The Marathon source includes extraction tools (`shapeextract.c`, `sndextract.c`) that read shapes/sounds from resource forks. This creates confusion:

**What happened historically**:
1. Original Marathon (1994) shipped with shapes/sounds in resource forks
2. Bungie provided extraction tools to convert them to data fork binary files
3. Marathon 2 retail uses the **extracted versions** (data fork files)
4. The comment in `shapes_macintosh.c:53` saying "open the resource fork" is **misleading** - it actually opens the data fork!

**For your port**:
- If you have retail Marathon 2 data files, Shapes/Sounds are already in data fork format
- You can read them with standard C file I/O on any platform
- **No resource fork parsing needed for core gameplay!**

**Only the Images file needs resource fork extraction**, and it's optional (just for interface graphics).

See [Chapter 31: Resource Forks](chapters/31_resource_forks.md) for detailed extraction instructions if needed.

---

## DOCUMENTATION CROSS-REFERENCE

When you need specific information, use this quick lookup:

| Topic | Primary Chapter | Source Files |
|-------|-----------------|--------------|
| **Architecture overview** | [Ch 1-3](chapters/01_introduction.md) | CLAUDE.md |
| **Fixed-point math** | [App D](chapters/appendix_d_fixedpoint.md) | cseries.h |
| **World/polygons** | [Ch 4](chapters/04_world.md) | map.h, map_constructors.c |
| **Rendering pipeline** | [Ch 5](chapters/05_rendering.md) | render.c |
| **Physics/collision** | [Ch 6](chapters/06_physics.md) | physics.c, world.c |
| **Game loop** | [Ch 7](chapters/07_game_loop.md) | marathon2.c |
| **Monsters/AI** | [Ch 8](chapters/08_entities.md) | monsters.c, monster_definitions.h |
| **File formats** | [Ch 10](chapters/10_file_formats.md) | wad.c, shapes.c |
| **Performance** | [Ch 11](chapters/11_performance.md) | scottish_textures.c |
| **Sound system** | [Ch 13](chapters/13_sound.md) | game_sound.c |
| **cseries.lib types** | [Ch 24](chapters/24_cseries.md) | cseries.h |
| **Resource forks** | [Ch 31](chapters/31_resource_forks.md) | - |
| **Frame walkthrough** | [Ch 32](chapters/32_frame.md) | (all systems) |
| **Constants/limits** | [App B](chapters/appendix_b_reference.md) | map.h, render.h |
| **Source file index** | [App C](chapters/appendix_c_files.md) | - |

**This is an achievable port! The Marathon engine is surprisingly clean and most code is portable C. Good luck!** 🎮
