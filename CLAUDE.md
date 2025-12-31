# Claude Conversation Summary: Marathon 2 Source Code Analysis & Porting Plan

**Date**: 2025-12-23 (Updated: 2025-12-31)
**Context**: Analysis of Marathon 2: Durandal and Marathon Infinity source code, with focus on porting to modern platforms using the Full Beans framework

---

## 📚 Documentation Index

| Document | Purpose |
|----------|---------|
| **Docs/** | Complete tutorial-style documentation (32 chapters + 4 appendices). Teaches engine concepts progressively in "Crafting Interpreters" style. **Primary reference for all technical details.** |
| **CLAUDE.md** (this file) | Porting-focused guide. Platform abstraction strategies, Mac API replacements, and practical porting decisions. |
| **Docs/porting_progress.md** | Step-by-step porting plan with 12 milestones, checkboxes, and chapter references. |

### Chapter Files (Tutorial Style) - ALL COMPLETE

The `Docs/` directory contains tutorial-style documentation that teaches engine concepts progressively. See `Docs/README.md` for the full table of contents.

| Section | Chapters | Key Topics |
|---------|----------|------------|
| **Foundation** | 1-3 | Introduction, source organization, engine overview |
| **Core Engine** | 4-6 | World representation, rendering, physics |
| **Game Systems** | 7-9 | Game loop, entities/AI, networking |
| **Data and I/O** | 10-13 | File formats, optimization, data structures, sound |
| **Game Mechanics** | 14-20 | Items, controls, damage, multiplayer, RNG, animation, automap |
| **Interface & Effects** | 21-24 | HUD, fades, camera, cseries.lib |
| **Environment Systems** | 25-29 | Media/liquids, effects, scenery, terminals, music |
| **Engine Internals** | 30-32 | Error handling, resource forks, life of a frame |
| **Appendices** | A-D | Glossary, quick reference, source index, fixed-point |

**For learning how the engine works**: Start with `Docs/04_world.md` → `05_rendering.md` → `06_physics.md`
**Essential for porting**: `Docs/32_frame.md` (Life of a Frame) shows complete frame lifecycle
**For quick reference**: See `Docs/appendix_b_reference.md`
**For porting tasks**: See `Docs/porting_progress.md`

---

## ⚠️ CRITICAL DISCOVERY: Resource Fork Clarification

**IMPORTANT**: There's widespread confusion about Marathon's file formats. Here's the truth:

**Files you CAN read directly (standard C file I/O)**:
- ✓ Shapes16/Shapes8 - Data fork binary files
- ✓ Sounds16/Sounds8 - Data fork binary files
- ✓ Map files - Marathon WAD format
- ✓ Saved games - Marathon WAD format

**Files that use resource forks (optional)**:
- ✗ Images file - PICT graphics (can stub or pre-extract)
- ✗ Scenario files - Custom campaigns (can skip)
- ✗ Music - QuickTime (can skip)

**Bottom line**: You can port Marathon 2 without touching resource forks at all! Only the Images file (interface graphics) uses them, and that's optional.

> **For complete resource fork documentation**: See `Docs/31_resource_forks.md` - includes binary format details, extraction strategies, and decision trees for porting.

---

## Conversation Overview

This conversation explored the Marathon 2 source code (classic 1994 FPS by Bungie) and developed a comprehensive strategy for porting it to modern cross-platform systems using the Full Beans/Fenster windowing framework.

### What We Covered

1. **Codebase Explanation** - Comprehensive analysis of Marathon 2's architecture
2. **Rendering System Deep Dive** - Portal-based rendering vs Wolfenstein raycasting, projection math, clipping, overdraw
3. **Porting Strategy** - Complete plan with 12 milestones for porting to Full Beans
4. **Mac Memory Management** - Understanding handles and how to replace them with modern pointers
5. **File Format Specifications** - Detailed documentation of Marathon's data file formats
6. **Tutorial Documentation** - 32 chapters + 4 appendices in "Crafting Interpreters" style

---

## Key Findings

### Marathon 2 Architecture

**What it is**:
- Original C source code for Marathon 2: Durandal (GPL 2, released 2000) and Marathon Infinity (GPL 3, released 2011)
- Classic first-person shooter with software 3D rendering
- Originally for Macintosh (68K and PowerPC architectures)

**Structure**:
- `marathon_src/marathon2/` - Main game code (~68,000 lines of C, 78 files)
- `marathon_src/cseries.lib/` - Shared utilities (~4,400 lines)
- Built for Classic Mac OS using MPW (Macintosh Programmer's Workshop)

**Key Technologies**:
- Pure C with some 68K/PowerPC assembly for performance
- QuickDraw for graphics
- Sound Manager for audio
- AppleTalk for networking
- Fixed-point math throughout (no floating-point)

**Core Components**:
- **Rendering**: Portal-based visibility culling, software texture mapping
- **Physics**: Fixed-point collision detection and movement
- **Map System**: Polygon-based geometry with dynamic platforms
- **Game Logic**: 47 monster types, weapons, items, 30 ticks/sec loop
- **Networking**: Peer-to-peer multiplayer with deterministic simulation
- **File Formats**:
  - Marathon WAD files (custom format, NOT Doom WADs) for maps
  - Data fork binary files for shapes and sounds (readable on all platforms!)
  - Resource forks ONLY for Images file (optional)

---

## Rendering System Overview

> **For complete details**: See `Docs/05_rendering.md`
> **For frame lifecycle**: See `Docs/32_frame.md`

**Quick summary**:
- Portal-based visibility culling (50-100 polygons rendered from 500-1000 total)
- Two texture mappers: affine for floors/ceilings, perspective-correct for walls
- Pre-computed shading tables for lighting
- Main file: `render.c` (3,879 lines) - 99% platform-independent!

**Key topics covered in the rendering chapter**:
- How Marathon differs from Wolfenstein's DDA raycasting (portal projection vs ray-wall intersection)
- Wall endpoint projection and portal clipping (with diagrams)
- Overdraw and the painter's algorithm (no Z-buffer)
- Render order within polygons: ceiling → walls → floor → objects
- Source-verified implementation details from `render.c`

---

## Porting Strategy Summary

### Target Platform: Full Beans / Fenster

**What it provides**:
- 32-bit ARGB framebuffer with direct pixel access
- Cross-platform window/input (macOS, Windows, Linux)
- Single-header library (fenster.h)
- Minimal dependencies

**Perfect fit because**:
- Marathon uses software rendering → writes directly to framebuffer
- Simple API matches Marathon's needs
- Cross-platform without heavyweight frameworks

### Porting Assessment

**Good news**:
- Core renderer (render.c) is ~99% platform-independent
- Game logic (physics, monsters, weapons) is portable C
- Only ~11 Mac-specific files need replacement
- Map files (Marathon WAD format) are platform-independent

**Note on file formats**:
- Marathon uses its own "WAD" format ONLY for maps (not like Doom WADs)
- Shapes and sounds are in **data fork** binary files, readable on all platforms!
- Resource forks are ONLY used for Images file (not essential) and optional scenarios
- No Mac Resource Manager needed for core gameplay files!

**Challenges**:
- Replace Mac APIs (QuickDraw, Sound Manager, File Manager)
- Convert 8-bit palette mode to 32-bit ARGB
- Replace assembly optimizations with C fallbacks
- Handle big-endian byte order (Mac) on little-endian systems (x86)
- Extract sound data from sound files
- Add audio library (miniaudio suggested)

### The Plan: 12 Milestones

**Phase 1: Foundation** (6-12 hours)
- M1: Project setup & build system
- M2: Platform abstraction layer (memory, files, timing, graphics)

**Phase 2: Data Loading** (18-26 hours)
- M3: File loading system (Marathon WADs for maps, binary file reader for shapes/sounds)
- M4: Shape/texture loading with color conversion from data fork files

**Phase 3: Minimal Rendering** (20-32 hours)
- M5: Input system (keyboard/mouse → action flags)
- M6: Graphics bridge (framebuffer setup)
- M7: Render first frame (the big milestone!)

**Phase 4: Playable Game** (28-40 hours)
- M8: Game loop integration (30 ticks/sec)
- M9: Combat & entities (monsters, weapons, items)
- M10: Basic audio (miniaudio integration)

**Phase 5: Polish** (10-18 hours)
- M11: HUD & interface
- M12: Performance optimization

**Total estimate**: 78-124 hours (2-3 weeks full-time, 6-8 weeks part-time)

---

## Key Deliverables

### Files Created

**`Docs/porting_progress.md`** - Complete porting plan with:
- Detailed tasks and checkboxes for all 12 milestones
- Testing criteria for each milestone
- Code examples for platform abstractions
- Technical appendix with API references
- Color conversion notes
- Estimated effort breakdown

### Files to Reference

**Marathon source**:
- Location: `./marathon_src/marathon2/` and `./marathon_src/cseries.lib/` (this repository)
- Key files: `marathon_src/marathon2/render.c`, `marathon_src/marathon2/scottish_textures.c`, `marathon_src/marathon2/map.c`
- Mac-specific files: `*_macintosh.c` (11 files to replace)

**Full Beans framework**:
- Location: (configure path to your Full Beans installation)
- Key files: `fenster.h`, `renderer.c/h`, `main.c` (example)

---

## Next Steps

When resuming this project:

1. **Read `Docs/porting_progress.md`** - Your complete roadmap
2. **Start with Milestone 1** - Project setup
3. **Follow the critical path**: M1→M2→M3→M4→M5→M6→M7→M8→M9
4. **Quick validation points**:
   - After M7: Should see static 3D rendering
   - After M8: Should be able to walk around
   - After M9: Should be able to fight monsters

### Platform Abstraction Strategy

Create these modules in `src/platform/`:
- `memory.c` - Replace Mac Memory Manager (simple malloc/free wrappers)
- `files.c` - Replace File Manager (FSSpec, FSRead/Write)
- `endian.c` - Byte swapping utilities for big-endian data
- `timing.c` - Replace TickCount()
- `graphics.c` - Framebuffer abstraction
- `sound.c` - Replace Sound Manager (use miniaudio)
- `input.c` - Replace GetKeys/GetMouse (use Fenster)

### Mac Memory Management: Handles Explained

**What are handles?** Handles are pointer-to-pointer (`type**`) used for relocatable memory on classic Mac OS. Note that `cseries.h:130` defines `typedef void *handle` (single pointer) as an abstraction, but actual usage in Marathon employs double pointers (`type**`) for collections and shading tables.

**Why Marathon used them:**
- Classic Mac OS (System 6/7) had no virtual memory on 68K Macs
- Heap fragmentation was a critical problem
- Handles allowed the OS to move memory blocks around to compact the heap
- The handle (pointer-to-pointer) stays at a fixed address while the actual data moves

**What Marathon uses handles for:**
1. **Shape collections** (`shape_definitions.h:17`) - Large sprite/texture data (hundreds of KB per collection)
   ```c
   struct collection_definition **collection;  // Handle to loaded graphics
   ```
2. **Shading tables** (`shape_definitions.h:18`) - Pre-computed lighting lookup tables
   ```c
   void **shading_tables;  // Handle to lighting data
   ```

**How handles work on Mac:**
```c
// Allocate relocatable memory
Handle h = NewHandle(1024);

// Lock it before accessing (pins it in place)
HLock(h);
byte* data = *h;  // Dereference to get actual pointer
data[0] = 42;
HUnlock(h);       // Allow OS to move it again
```

**For porting - Replace with regular pointers:**
Modern OSes have virtual memory, so handles are unnecessary. Simply:
- Change `type**` to `type*` in structure definitions (e.g., `shape_definitions.h`)
- Replace `NewHandle(size)` with `malloc(size)`
- Replace `HLock(h)` / `HUnlock(h)` / `MoveHHi(h)` with no-ops (not needed)
- Replace `DisposeHandle(h)` with `free(h)`
- Remove one level of pointer indirection: `*handle` becomes `handle`
- Update all code that accesses these structures accordingly

Example transformation:
```c
// Original Mac code (shapes_macintosh.c):
struct collection_definition **collection;
collection = (struct collection_definition **) NewHandle(size);
HLock((Handle)collection);
struct collection_definition* data = *collection;  // Dereference
// ... use data

// Modern replacement:
struct collection_definition *collection;
collection = (struct collection_definition *) malloc(size);
// ... use collection directly (no dereference needed)
```

### Critical Decisions Made

1. **Framebuffer format**: Render to 32-bit ARGB (simpler than 16-bit conversion)
2. **Assembly replacement**: Use C fallbacks (already exist in codebase)
3. **Audio library**: Recommended miniaudio (single-header, cross-platform)
4. **Networking**: Stub out for single-player port (can add later)
5. **Byte order**: Implement byte swapping for big-endian data on x86

---

## Technical Notes

### Color Conversion

Marathon uses 16-bit RGB555 color tables:
```c
struct rgb_color {
  uint16 red, green, blue;  // 0-65535 range
};
```

Convert to 32-bit ARGB:
```c
uint32_t marathon_to_argb(struct rgb_color* c) {
  uint8_t r = c->red >> 8;
  uint8_t g = c->green >> 8;
  uint8_t b = c->blue >> 8;
  return 0xFF000000 | (r << 16) | (g << 8) | b;
}
```

**Important**: Pre-convert all shading tables to avoid per-pixel conversion overhead.

### Marathon File Formats Summary

> **For complete specifications**: See `Docs/10_file_formats.md`

Marathon uses three types of data files:

| File Type | Format | Readable on All Platforms? |
|-----------|--------|---------------------------|
| Map files | Marathon WAD (custom, NOT Doom) | ✓ Yes - standard fopen/fread |
| Shapes (Shapes8/16) | Data fork binary | ✓ Yes - standard fopen/fread |
| Sounds (Sounds8/16) | Data fork binary | ✓ Yes - standard fopen/fread |
| Images | Mac resource fork | ✗ No - needs extraction or can stub |

**Key insight**: Despite Marathon being a classic Mac game, the core game data (shapes, sounds, maps) is all in standard binary formats. Only optional interface graphics use Mac-specific resource forks.

**Byte order warning**: All Marathon data is big-endian. Need byte swapping on x86:
```c
uint16_t swap16(uint16_t val) {
    return (val << 8) | (val >> 8);
}

uint32_t swap32(uint32_t val) {
    return ((val & 0xFF) << 24) | ((val & 0xFF00) << 8) |
           ((val & 0xFF0000) >> 8) | ((val & 0xFF000000) >> 24);
}
```

---

## Resource Forks Reference

> **Complete documentation**: See `Docs/31_resource_forks.md`
>
> Includes:
> - Detailed binary format specifications with diagrams
> - File-by-file analysis of what uses resource forks
> - Extraction strategies and tools
> - Decision trees for porting strategy

**Quick summary**: For core gameplay porting, you don't need resource forks at all. Shapes, Sounds, and Maps are all standard binary files readable with `fopen`/`fread`.

### Fenster API Quick Reference

```c
// Window setup
struct fenster f = {.title = "Marathon 2", .width = 640, .height = 480};
uint32_t buffer[640 * 480];
f.buf = buffer;

// Main loop
fenster_open(&f);
while (fenster_loop(&f) == 0) {
  // Input: f.keys[], f.x, f.y, f.mouse
  // Render: f.buf[y * 640 + x] = 0xAARRGGBB
}
fenster_close(&f);
```

### File Organization Strategy

Recommended structure:
```
marathon-port/
├── src/
│   ├── platform/      # Full Beans + platform abstraction
│   ├── marathon/      # Marathon 2 source (copied, ~60 files)
│   └── main.c         # New entry point
├── data/              # Map WADs and data fork files (Shapes, Sounds)
└── Makefile
```

**Note**: Marathon map files use Marathon's custom WAD format. Shapes and Sounds are data fork binary files, readable with standard file I/O on any platform. Only the optional Images file uses resource forks.

---

## Why This Port Is Feasible

1. **Clean separation**: Rendering is 99% platform-independent
2. **Well-structured**: Clear module boundaries (render, map, physics, etc.)
3. **Proven portability**: Aleph One project successfully ported Marathon to SDL
4. **Modern hardware**: Even software rendering is fast enough on modern CPUs
5. **Simple target**: Fenster provides exactly what we need, nothing more

---

## Historical Context

Marathon 2 was released in 1995 by Bungie (before Halo). The rendering techniques were advanced for the time:
- Portal rendering (before Quake popularized BSP trees)
- Deterministic networking (innovative for multiplayer)
- Software 3D that ran on 68040 Macs at 15-30 FPS

The source release in 2000/2011 was a gift to the community, enabling projects like Aleph One (the modern open-source engine).

---

## Questions for Next Session

If you have questions when resuming:

1. **Build issues**: Check compiler flags, Mac header guards
2. **Rendering bugs**: Verify texture loading, shading tables, color conversion
3. **Performance issues**: Profile first, optimize texture mapper, pre-convert colors
4. **Mac dependencies**: Search for QuickDraw calls, replace with platform layer

---

## Conclusion

This is a **highly feasible port**. The Marathon engine is surprisingly clean, with excellent separation between platform-specific and game logic. The core 3D renderer is a gem of 1990s software engineering that can run on modern systems with minimal changes.

**Estimated success probability**: Very high
**Biggest challenge**: Time and persistence, not technical difficulty
**Most exciting milestone**: M7 - seeing the first rendered 3D frame

Good luck with the port! 🎮
