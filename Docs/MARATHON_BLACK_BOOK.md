# Marathon Engine Black Book
## A Technical Deep-Dive into Marathon 2 & Infinity Game Engine

**Author**: Technical analysis based on Marathon Infinity source code (GPL v3)
**Version**: 1.3
**Date**: December 2025
**Source**: `./marathon2/` and `./cseries.lib/` (this repository)
**Note**: This analysis is based on the Marathon Infinity source release (2011), which represents the most complete and refined version of the Marathon 2 engine.

---

## Table of Contents

1. [Introduction](#1-introduction)
   - [About Marathon 2 & Infinity](#about-marathon-2--infinity)
   - [Source Code Statistics](#source-code-statistics)
   - [About This Document](#about-this-document)
   - [Marathon 2 vs. Infinity: What Changed?](#marathon-2-vs-infinity-what-changed)
2. [Source Code Organization](#2-source-code-organization)
   - [Directory Structure](#directory-structure)
   - [File Categories](#file-categories)
   - [Platform Abstraction](#platform-abstraction)
   - [Header Dependencies](#header-dependencies)
3. [Engine Overview](#3-engine-overview)
4. [World Representation](#4-world-representation)
5. [Rendering System](#5-rendering-system)
6. [Physics and Collision](#6-physics-and-collision)
7. [Game Loop and Timing](#7-game-loop-and-timing)
8. [Entity Systems](#8-entity-systems)
   - [Motion Sensor System](#motion-sensor-system)
   - [Terminal System](#terminal-system-computer-interface)
   - [Replay/Recording System](#replayrecording-system)
   - [Save/Load System](#saveload-system)
9. [Networking Architecture](#9-networking-architecture)
10. [File Formats](#10-file-formats)
11. [Performance and Optimization](#11-performance-and-optimization)
12. [Appendix: Data Structures](#12-appendix-data-structures)
13. [Sound System](#13-sound-system)
14. [Items & Inventory System](#14-items--inventory-system)
15. [Control Panels System](#15-control-panels-system)
16. [Damage System](#16-damage-system)
17. [Multiplayer Game Types](#17-multiplayer-game-types)
18. [Random Number Generation](#18-random-number-generation)
19. [Shape Animation System](#19-shape-animation-system)
20. [Automap/Overhead Map System](#20-automapoverhead-map-system)
21. [HUD Rendering System](#21-hud-rendering-system)
22. [Screen Effects & Fades](#22-screen-effects--fades)
23. [View Bobbing & Camera System](#23-view-bobbing--camera-system)
24. [cseries.lib Utility Library](#24-cserieslib-utility-library)
25. [Media/Liquid System](#25-medialiquid-system)
26. [Visual Effects System](#26-visual-effects-system)
27. [Scenery Objects](#27-scenery-objects)
28. [Computer Terminal System](#28-computer-terminal-system)
29. [Music/Soundtrack System](#29-musicsoundtrack-system)
30. [Error Handling & Progress Display](#30-error-handling--progress-display)
31. [Resource Forks: Complete Guide](#31-resource-forks-complete-guide)
- [Appendix A: Glossary of Terms](#appendix-a-glossary-of-terms)
- [Appendix B: Quick Reference Card](#appendix-b-quick-reference-card)
- [Appendix C: Source File Index](#appendix-c-source-file-index)
- [Appendix D: Fixed-Point to Floating-Point Conversion](#appendix-d-fixed-point-to-floating-point-conversion-optional-modernization)

---

## 1. Introduction

### About Marathon 2 & Infinity

Marathon 2: Durandal and Marathon Infinity represent the pinnacle of Bungie's classic FPS trilogy, both built on the same sophisticated game engine.

**Marathon 2: Durandal**:
- Released: November 24, 1995
- Developer: Bungie Software
- Platform: Macintosh (68K and PowerPC)
- Source Release: January 2000 (GPL v2, incomplete)

**Marathon Infinity**:
- Released: October 15, 1996
- Developer: Bungie Software
- Built on Marathon 2 engine with refinements
- Source Release: July 2011 (GPL v3, comprehensive)

**This Document's Basis**:
This Black Book analyzes the Marathon Infinity source code (2011 release), which is the most complete and well-preserved version of the engine. The Infinity release includes the full codebase without the commercial redactions present in the 2000 Marathon 2 release. The engine architecture described here applies to both games, with Infinity representing the refined and final iteration.

**Technical Achievements** (Common to Both Games):
- Full 3D environments with varying floor/ceiling heights
- True room-over-room geometry
- Portal-based visibility culling
- Deterministic peer-to-peer networking (up to 8 players)
- 30 Hz fixed-timestep physics
- Software texture mapping without FPU requirements

### Source Code Statistics

**Code Metrics**:
- ~68,000 lines of C code (marathon2/)
- ~4,400 lines of utility code (cseries.lib/)
- 78 source files
- Built with MPW (Macintosh Programmer's Workshop)

**Key Files**:
- `render.c` (3,879 lines) - Rendering pipeline
- `map.c` (3,456 lines) - World representation
- `physics.c` (2,234 lines) - Movement and collision
- `scottish_textures.c` (1,458 lines) - Texture mapper
- `monsters.c` - AI and entity behavior
- `weapons.c` - Combat system

### About This Document

This "Black Book" provides comprehensive technical analysis of the Marathon game engine, inspired by Fabien Sanglard's excellent work on Doom and Wolfenstein 3D. The goal is to document the engineering solutions that made Marathon possible on mid-1990s hardware.

### Marathon 2 vs. Infinity: What Changed?

While Marathon Infinity used the same core engine as Marathon 2, Bungie made several refinements and additions:

#### Engine & Code Improvements

**Source Code Completeness**:
- Marathon 2 source (2000): Edited for release, with serial number generation and some `cseries.lib` code removed
- Marathon Infinity source (2011): Comprehensive, unredacted release matching the commercial product
- Files edited in M2 release: `game_dialogs.c`, `game_wad.c`, `interface.c`, `player.c`, `preferences.c`, `shell.c`

**Physics Models**:
- **Multiple physics models**: Infinity introduced selectable physics models that could be specified per-level
- Physics variations affected player movement speeds, weapon behavior, and game balance
- Maps could specify which physics model to use via the `static_data.physics_model` field
- This allowed scenario designers to fine-tune gameplay without engine modifications

**Networking Enhancements**:
- Refined netcode for better synchronization
- Improved lag compensation
- Better handling of packet loss

#### Gameplay Features

**New Game Types**:
- Added specialized multiplayer modes beyond M2's offerings
- Enhanced team-based gameplay options
- Improved scoring and statistics tracking

**Level Design Capabilities**:
- Support for more complex trigger systems
- Enhanced scripting possibilities through terminals
- More sophisticated platform (elevator/door) behaviors
- Ambient sound improvements

**Visual Enhancements**:
- Refined texture mapping (same algorithm, better assets)
- Improved lighting effects
- Enhanced particle effects

#### Content & Scenarios

**The Major Difference**:
The biggest distinction between M2 and Infinity wasn't the engine—it was the **scenario system**:
- Marathon Infinity shipped with a powerful level editor (Forge) and physics editor (Anvil)
- Included multiple complete scenarios: main campaign + two additional campaigns ("Blood Tides of Lh'owon", "Evil")
- Pioneered the concept of user-generated content for FPS games
- Community could create and share custom maps, physics, and complete campaigns

**File Format Evolution**:
- Enhanced map file format to support new features
- Backward compatible with M2 maps
- New tag types for additional functionality
- Physics model definitions stored in separate files

#### Technical Architecture

**Core Engine Unchanged**:
- Same portal-based rendering system
- Same fixed-point mathematics
- Same 30 Hz fixed timestep
- Same polygon-based world representation
- Same deterministic networking model

**Refinements**:
- Better memory management
- Optimized texture mapping inner loops
- Improved AI pathfinding
- More efficient object spawning

#### Source Code Statistics Comparison

| Metric | Marathon 2 (2000) | Marathon Infinity (2011) |
|--------|-------------------|--------------------------|
| License | GPL v2 | GPL v3 |
| Completeness | Partial (edited) | Complete |
| Total Lines | ~68,000 (estimated) | ~72,000+ |
| Key Files | Many redacted | Fully intact |
| Physics Models | 1 (hardcoded) | Multiple (data-driven) |

**Why Infinity Source Matters**:
The Infinity source release is the definitive reference for understanding Marathon's engine because:
1. It's complete and unredacted
2. It represents the final, most refined version
3. It includes all the physics model variations
4. It's what modern ports (Aleph One) are based on

**Implication for This Document**:
All technical details, code examples, and architectural descriptions in this Black Book are based on the Infinity source. These details apply equally to Marathon 2, as Infinity preserved the core engine architecture while adding features through the data-driven physics system rather than fundamental engine changes.

---

## 2. Source Code Organization

### Directory Structure

The Marathon Infinity source code is organized into two main directories:

```
m2-infinity-source-code-main/
├── marathon2/              # Main game code (~68,000 lines)
│   ├── *.c, *.h           # Core game source files (75 .c, 71 .h)
│   ├── *.a                # 68K assembly files
│   ├── *.s                # PowerPC assembly files
│   ├── editor code/       # Level editor stubs
│   ├── extract/           # Data extraction tools
│   │   ├── shapeextract.c # Extract shapes from resource fork
│   │   └── sndextract.c   # Extract sounds from resource fork
│   ├── buildprogram       # MPW build script
│   └── *.make             # MPW makefile fragments
│
├── cseries.lib/            # Shared utility library (~4,400 lines)
│   ├── cseries.h          # Core types and macros (platform-independent)
│   ├── macintosh_cseries.h # Mac-specific extensions
│   ├── byte_swapping.*    # Endianness utilities
│   ├── checksum.*         # CRC calculations
│   ├── rle.*              # Run-length encoding
│   └── *.c                # Utility implementations
│
├── licenses/               # GPL license files
│   ├── Marathon 2 Source Code License.txt    # GPL v2
│   └── Infinity Source Code License.txt      # GPL v3
│
└── README.md               # Original source release notes
```

### File Categories

The source files can be categorized by their function and platform dependency:

#### Core Game Logic (Platform-Independent)

These files contain the heart of the game and require minimal changes for porting:

| File | Lines | Purpose |
|------|-------|---------|
| `render.c` | 3,879 | Portal-based 3D rendering |
| `map.c` | 3,456 | World geometry and queries |
| `physics.c` | 2,234 | Movement, collision, gravity |
| `monsters.c` | ~3,000 | AI, behavior, combat |
| `weapons.c` | ~2,500 | Weapon logic, firing, ammo |
| `player.c` | ~2,000 | Player state, controls |
| `projectiles.c` | ~1,500 | Bullet/missile physics |
| `platforms.c` | ~1,200 | Elevators, doors, switches |
| `effects.c` | ~800 | Particles, explosions |
| `items.c` | ~600 | Pickups, powerups |
| `lightsource.c` | ~500 | Dynamic lighting |
| `media.c` | ~400 | Water, lava physics |
| `flood_map.c` | ~900 | Zone/area calculations |
| `pathfinding.c` | ~800 | Monster navigation |
| `scenery.c` | ~300 | Static objects |
| `world.c` | ~400 | World queries |

#### Data Management (Mostly Portable)

| File | Purpose | Porting Notes |
|------|---------|---------------|
| `wad.c` | Marathon WAD file format | Replace Mac file I/O |
| `game_wad.c` | Level loading/saving | Replace Mac file I/O |
| `shapes.c` | Shape collection management | Needs platform layer |
| `textures.c` | Texture management | Platform-independent |
| `game_sound.c` | Sound playback logic | Needs audio backend |

#### Rendering Pipeline (Mostly Portable)

| File | Purpose | Porting Notes |
|------|---------|---------------|
| `scottish_textures.c` | Software texture mapping | C fallbacks exist |
| `low_level_textures.c` | Texture utilities | Platform-independent |
| `screen.c` | Screen management | Replace QuickDraw |
| `fades.c` | Screen transitions | Replace palette code |
| `overhead_map.c` | Automap rendering | Replace drawing calls |

#### Mac-Specific Files (Must Replace)

These 11 files contain Macintosh-specific code and need platform replacements:

| File | Lines | Mac APIs Used | Replacement Strategy |
|------|-------|---------------|---------------------|
| `files_macintosh.c` | ~430 | FSSpec, FSRead/Write | stdio (fopen, fread) |
| `shapes_macintosh.c` | ~660 | FSSpec, Handles | stdio + malloc |
| `sound_macintosh.c` | ~1,000 | Sound Manager | miniaudio/SDL_mixer |
| `vbl_macintosh.c` | ~400 | VBL interrupts, GetKeys | Fenster input polling |
| `interface_macintosh.c` | ~1,700 | Dialogs, Menus | Custom or stub |
| `game_window_macintosh.c` | ~220 | GrafPorts, Regions | Framebuffer direct |
| `overhead_map_macintosh.c` | ~200 | QuickDraw lines | Line drawing to buffer |
| `wad_macintosh.c` | ~150 | FSSpec path handling | POSIX paths |
| `wad_prefs_macintosh.c` | ~200 | Preferences file | Simple config file |
| `preprocess_map_mac.c` | ~150 | Resource fork access | Not needed |
| `mouse.c` | ~200 | GetMouse, CursorDevice | Fenster mouse state |

#### Network Code (Can Stub for Single-Player)

| File | Purpose | Notes |
|------|---------|-------|
| `network.c` | Core networking | AppleTalk-based |
| `network_ddp.c` | DDP protocol | Mac-specific |
| `network_adsp.c` | ADSP streams | Mac-specific |
| `network_modem.c` | Modem support | Mac-specific |
| `network_games.c` | Game types | Portable logic |
| `network_dialogs.c` | UI | Mac-specific |

#### Assembly Files (Have C Fallbacks)

| File | Architecture | Purpose |
|------|--------------|---------|
| `scottish_textures.a` | 68K | Optimized texture inner loops |
| `scottish_textures.s` | PowerPC | Optimized texture inner loops |
| `scottish_textures16.a` | 68K | 16-bit texture mapping |
| `screen.a` | 68K | Screen blitting |
| `quadruple.s` | PowerPC | 64-bit math |
| `network_listener.a` | 68K | Network interrupt handler |
| `cseries.a` | 68K | Utility functions |

**Note**: All assembly-optimized functions have C equivalents in the same source files, controlled by `#ifdef` blocks.

#### Definition Headers (Data Tables)

| Header | Purpose |
|--------|---------|
| `monster_definitions.h` | 47 monster type definitions |
| `weapon_definitions.h` | Weapon stats and behavior |
| `projectile_definitions.h` | Projectile types |
| `effect_definitions.h` | Visual effect types |
| `item_definitions.h` | Pickup items |
| `platform_definitions.h` | Platform/door types |
| `sound_definitions.h` | Sound effect IDs |
| `scenery_definitions.h` | Scenery object types |

### Platform Abstraction

Marathon's codebase has a clear (though sometimes implicit) separation between portable and platform-specific code.

#### The cseries.lib Foundation

The `cseries.lib` directory provides the abstraction layer:

```c
// cseries.h - Platform-independent definitions
typedef long fixed;           // 16.16 fixed-point
typedef unsigned short word;
typedef unsigned char byte;
typedef byte boolean;

#define FIXED_ONE (1<<16)     // 65536
#define TRUE 1
#define FALSE 0
#define NONE -1

// Memory management (redirects to platform layer)
#define malloc(size) new_pointer(size)
#define free(ptr) dispose_pointer(ptr)
void *new_pointer(long size);
void dispose_pointer(void *pointer);

// Debug support
void assert(expr);
void halt();
int dprintf(const char *format, ...);
```

```c
// macintosh_cseries.h - Mac-specific extensions
#include <Memory.h>
#include <QuickDraw.h>
#include <Events.h>
// ... Mac toolbox headers

// Mac handle wrappers
Handle NewHandle(Size size);
void DisposeHandle(Handle h);
void HLock(Handle h);
void HUnlock(Handle h);
```

#### Platform Abstraction Pattern

Marathon uses a consistent pattern for platform abstraction:

```
┌─────────────────────────────────────────────────────────────┐
│                    Game Logic Layer                          │
│  render.c, physics.c, monsters.c, weapons.c, etc.           │
│  (Platform-independent, uses abstract interfaces)            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Abstract Interface                         │
│  shapes.c, game_sound.c, vbl.c, screen.c                    │
│  (Defines interface, may have some portable code)            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Platform Implementation                      │
│  shapes_macintosh.c, sound_macintosh.c, vbl_macintosh.c     │
│  (Mac-specific code, replace for porting)                    │
└─────────────────────────────────────────────────────────────┘
```

#### Key Abstractions to Replace

**1. Memory Management**

```c
// Original (Mac handles for relocatable memory)
Handle h = NewHandle(size);
HLock(h);
void* ptr = *h;
// ... use ptr
HUnlock(h);
DisposeHandle(h);

// Replacement (modern systems don't need handles)
void* ptr = malloc(size);
// ... use ptr
free(ptr);
```

**2. File I/O**

```c
// Original (Mac File Manager)
FSSpec spec;
short refNum;
FSMakeFSSpec(vRefNum, dirID, name, &spec);
FSpOpenDF(&spec, fsRdPerm, &refNum);
FSRead(refNum, &count, buffer);
FSClose(refNum);

// Replacement (stdio)
FILE* fp = fopen(path, "rb");
fread(buffer, 1, count, fp);
fclose(fp);
```

**3. Timing**

```c
// Original (Mac Tick Count - 60 Hz)
unsigned long ticks = TickCount();

// Replacement
uint32_t ms = platform_get_ticks();  // milliseconds
```

**4. Input**

```c
// Original (Mac Events)
KeyMap keys;
GetKeys(keys);
Point mouse;
GetMouse(&mouse);

// Replacement (Fenster)
if (fenster->keys[KEY_UP]) { /* ... */ }
int mouse_x = fenster->x;
int mouse_y = fenster->y;
```

**5. Graphics**

```c
// Original (QuickDraw)
GrafPtr port;
GetPort(&port);
SetRect(&rect, 0, 0, width, height);
CopyBits(src, dst, &srcRect, &dstRect, srcCopy, NULL);

// Replacement (direct framebuffer)
memcpy(fenster->buf, screen_buffer, width * height * 4);
```

#### Conditional Compilation

The codebase uses preprocessor conditionals for platform-specific code:

```c
#ifdef envppc
    // PowerPC-specific code
#else
    #ifdef env68k
        // 68K-specific code
    #else
        // Generic/fallback code
    #endif
#endif

#ifdef mpwc
    #pragma segment marathon  // MPW segment directive
#endif
```

For porting, define your own platform macro:

```c
#ifdef FENSTER_PORT
    // Your portable implementation
#else
    // Original Mac code
#endif
```

### Header Dependencies

Understanding header dependencies is crucial for porting. Here's the include hierarchy:

```
cseries.h                    ← Base types (fixed, word, byte, boolean)
    │
    ├── macintosh_cseries.h  ← Mac toolbox (for original Mac build)
    │
    └── [Your platform header] ← Your platform layer (for porting)

map.h                        ← World structures (polygon, line, side)
    ├── world.h              ← World coordinate types
    └── shape_descriptors.h  ← Shape/texture references

render.h                     ← Rendering structures
    ├── map.h
    └── scottish_textures.h  ← Texture mapping

monsters.h                   ← Monster structures
    ├── map.h
    ├── monster_definitions.h
    └── effects.h

weapons.h                    ← Weapon structures
    ├── projectiles.h
    └── weapon_definitions.h
```

#### Minimal Include Set for Porting

To get the game compiling, you need these headers in order:

1. `cseries.h` - Base types and macros
2. `map.h` - World representation
3. `player.h` - Player structures
4. `render.h` - Rendering interface
5. `interface.h` - Game states

Then add others as needed based on compiler errors.

---

## 3. Engine Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│              Shell / Main Loop (30 Hz)              │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
      ┌────▼────┐                ┌───▼────┐
      │  Input  │                │Network │
      │ System  │                │  Sync  │
      └────┬────┘                └───┬────┘
           │                         │
           └────────┬────────────────┘
                    │
      ┌─────────────▼─────────────┐
      │   World Update (1 tick)   │
      │  • Lights                 │
      │  • Media                  │
      │  • Platforms              │
      │  • Players                │
      │  • Projectiles            │
      │  • Monsters               │
      │  • Effects                │
      └─────────────┬─────────────┘
                    │
           ┌────────▼────────┐
           │  Render Engine  │
           │ (Portal-Based)  │
           └─────────────────┘
```

### Core Design Principles

**1. Fixed-Point Math Everywhere**
```c
typedef long fixed;  // 16.16 fixed-point
#define FIXED_ONE (1<<16)           // 65536 = 1.0
#define WORLD_ONE 1024              // 10 fractional bits
#define WORLD_FRACTIONAL_BITS 10
```

No floating-point operations—crucial for:
- Deterministic behavior across platforms
- Performance on CPUs without FPUs
- Network consistency

> **🔧 For Porting:** Keep all fixed-point math as-is. The `fixed` type and macros in `cseries.h` are fully portable. Do NOT convert to floating-point—this would break network determinism and saved game compatibility.

**2. Fixed 30 Hz Timestep**
```c
#define TICKS_PER_SECOND 30
#define TICKS_PER_MINUTE 1800
```

Game logic runs at exactly 30 ticks/second:
- Physics deterministic
- Rendering decoupled
- Network-friendly

**3. Portal-Based Rendering**
- No BSP trees (unlike Doom/Quake)
- Recursive visibility through portals
- Typical: 50-100 polygons visible out of 500-1000 total

**4. Deterministic Simulation**
- Same inputs → same outputs
- Critical for networking
- Replay system possible

### Complete Subsystem Reference

This section provides a bird's-eye view of all Marathon engine subsystems, how they interact, and where to find detailed documentation.

#### Subsystem Interaction Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              MAIN LOOP (shell.c)                                │
│                           30 Hz fixed timestep                                   │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│    INPUT      │           │   NETWORK     │           │    AUDIO      │
│  vbl.c        │           │  network.c    │           │ game_sound.c  │
│  action_flags │──────────▶│  sync actions │           │ 3D positioned │
└───────────────┘           └───────┬───────┘           └───────▲───────┘
                                    │                           │
                                    ▼                           │
┌───────────────────────────────────────────────────────────────┼─────────────────┐
│                         WORLD UPDATE (marathon2.c)            │                 │
│                       update_world() - one tick               │                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ UPDATE ORDER (each tick):                                                │   │
│  │  1. update_lights()      → lightsource.c                                 │   │
│  │  2. update_medias()      → media.c (water/lava levels)                   │   │
│  │  3. update_platforms()   → platforms.c (doors/elevators)                 │   │
│  │  4. update_players()     → player.c + physics.c                          │   │
│  │  5. move_projectiles()   → projectiles.c                                 │   │
│  │  6. move_monsters()      → monsters.c + pathfinding.c                    │   │
│  │  7. update_effects()     → effects.c (particles/explosions)              │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                            │
│                                    ▼ (triggers sounds)                          │
└────────────────────────────────────┼────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           RENDERING (render.c)                                  │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌────────────┐  │
│  │   Portal    │───▶│   Polygon    │───▶│    Texture      │───▶│  Screen    │  │
│  │   Culling   │    │   Clipping   │    │    Mapping      │    │  Output    │  │
│  │  (map.c)    │    │  (render.c)  │    │(scottish_tex.c) │    │ (screen.c) │  │
│  └─────────────┘    └──────────────┘    └─────────────────┘    └────────────┘  │
│                                                                                 │
│  Uses: shapes.c (textures), lightsource.c (shading), fades.c (screen effects)  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              HUD OVERLAY                                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │  Motion Sensor  │  │   Health/Ammo   │  │  Weapon Sprite  │                 │
│  │ (game_window.c) │  │  (interface.c)  │  │   (weapons.c)   │                 │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### Data Flow Diagram

```
                    ┌─────────────────────────────────────┐
                    │          FILE SYSTEM                │
                    │  ┌─────────┐ ┌─────────┐ ┌───────┐ │
                    │  │Map WAD  │ │Shapes16 │ │Sounds │ │
                    │  └────┬────┘ └────┬────┘ └───┬───┘ │
                    └───────┼───────────┼──────────┼─────┘
                            │           │          │
                            ▼           ▼          ▼
                    ┌───────────┐ ┌──────────┐ ┌─────────────┐
                    │  wad.c    │ │ shapes.c │ │game_sound.c │
                    │game_wad.c │ │          │ │             │
                    └─────┬─────┘ └────┬─────┘ └──────┬──────┘
                          │            │              │
                          ▼            ▼              ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          RUNTIME DATA STRUCTURES                            │
│                                                                             │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────┐  ┌────────────────┐ │
│  │  map_polygons│  │ map_endpoints │  │  objects[]  │  │  players[]     │ │
│  │  [1024 max]  │  │ [2048 max]    │  │  [384 max]  │  │  [8 max]       │ │
│  └──────────────┘  └───────────────┘  └─────────────┘  └────────────────┘ │
│                                                                             │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────┐  ┌────────────────┐ │
│  │  monsters[]  │  │ projectiles[] │  │  effects[]  │  │  platforms[]   │ │
│  │  [220 max]   │  │ [32 max]      │  │  [64 max]   │  │  [64 max]      │ │
│  └──────────────┘  └───────────────┘  └─────────────┘  └────────────────┘ │
│                                                                             │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────┐  ┌────────────────┐ │
│  │  lights[]    │  │   medias[]    │  │   items[]   │  │  scenery       │ │
│  │  [64 max]    │  │ [16 max]      │  │  [64 max]   │  │  (in objects)  │ │
│  └──────────────┘  └───────────────┘  └─────────────┘  └────────────────┘ │
└────────────────────────────────────────────────────────────────────────────┘
```

#### Subsystem Quick Reference Table

| Subsystem | Primary Files | Purpose | Section |
|-----------|---------------|---------|---------|
| **World/Map** | map.c, world.c | Polygon geometry, endpoints, lines, sides | [§4](#4-world-representation) |
| **Rendering** | render.c, scottish_textures.c | Portal culling, texture mapping, 3D view | [§5](#5-rendering-system) |
| **Physics** | physics.c | Movement, gravity, collision detection | [§6](#6-physics-and-collision) |
| **Game Loop** | marathon2.c | 30 Hz update, state machine, tick processing | [§7](#7-game-loop-and-timing) |
| **Monsters** | monsters.c, pathfinding.c | AI states, behavior, navigation | [§8.1](#81-monster-system) |
| **Projectiles** | projectiles.c | Bullet/missile physics, hit detection | [§8.2](#82-projectile-system) |
| **Weapons** | weapons.c | Firing, reloading, dual-wielding, triggers | [§8.3](#83-weapon-system) |
| **Platforms** | platforms.c | Doors, elevators, crushers, triggers | [§8.4](#84-platform-system) |
| **Lighting** | lightsource.c | Dynamic lights, shading tables | [§5.4](#54-lighting-and-shading) |
| **Sound** | game_sound.c | 3D positioned audio, channels | [§13](#13-sound-system) |
| **Items** | items.c | Pickups, powerups, ammo | [§14](#14-items--inventory-system) |
| **Effects** | effects.c | Particles, explosions, debris | [§26](#26-visual-effects-system) |
| **Media** | media.c | Water, lava, sewage (liquids) | [§25](#25-medialiquid-system) |
| **Shapes** | shapes.c | Texture/sprite loading, collections | [§19](#19-shape-animation-system) |
| **HUD** | game_window.c, interface.c | Health, ammo, motion sensor | [§21](#21-hud-rendering-system) |
| **Automap** | overhead_map.c | Top-down map display | [§20](#20-automapoverhead-map-system) |
| **Terminals** | computer_interface.c | In-game computer screens | [§28](#28-computer-terminal-system) |
| **Network** | network.c | Peer-to-peer multiplayer sync | [§9](#9-networking-architecture) |
| **Files** | wad.c, game_wad.c | WAD format, level loading | [§10](#10-file-formats) |
| **Fades** | fades.c | Screen transitions, damage flash | [§22](#22-screen-effects--fades) |
| **Scenery** | scenery.c | Static decorative objects | [§27](#27-scenery-objects) |
| **Font/Text** | screen_drawing.c | 2D text rendering, fonts, UI drawing | [§21.4](#font-system) |

#### Subsystem Descriptions

**World/Map System** (`map.c`, `world.c`)
The foundation of everything. Stores the level geometry as interconnected polygons with endpoints, lines, and sides. Each polygon knows its neighbors, floor/ceiling heights, and textures. All spatial queries ("what polygon is this point in?") go through this system. The map is loaded from Marathon WAD files and remains static during gameplay (except for platform movements).
→ *Detailed in [Section 4: World Representation](#4-world-representation)*

**Rendering System** (`render.c`, `scottish_textures.c`)
Converts the 3D world into a 2D image using portal-based visibility culling. Starting from the player's polygon, it recursively renders through portal edges (lines shared between polygons). The `scottish_textures.c` module handles the actual pixel-level texture mapping with perspective correction for walls and affine mapping for floors/ceilings. Renders 50-100 polygons per frame from maps with 500-1000 total.
→ *Detailed in [Section 5: Rendering System](#5-rendering-system)*

**Physics System** (`physics.c`)
Handles all movement and collision. Uses fixed-point math exclusively for determinism. Objects move through the world with gravity, friction, and collision response. Players, monsters, and projectiles all share the same physics code with different parameters. Collision detection uses the polygon adjacency graph for efficiency.
→ *Detailed in [Section 6: Physics and Collision](#6-physics-and-collision)*

**Game Loop** (`marathon2.c`)
The heart of the engine. Runs at exactly 30 ticks per second. Each tick: processes input, updates all systems in a fixed order, handles network synchronization. The order matters: lights before media, platforms before players, players before monsters. `update_world()` is the main entry point.
→ *Detailed in [Section 7: Game Loop and Timing](#7-game-loop-and-timing)*

**Monster System** (`monsters.c`, `pathfinding.c`)
Manages 47 different creature types with sophisticated AI. Each monster has a state machine (idle, attacking, fleeing, etc.) and uses pathfinding to navigate the level. Monsters can fight each other (infighting), have different attack patterns, and respond to sound/sight triggers. The pathfinding system uses flood-fill zone calculations.
→ *Detailed in [Section 8.1: Monster System](#81-monster-system)*

**Projectile System** (`projectiles.c`)
Tracks bullets, missiles, grenades, and other moving projectiles. Each projectile has velocity, gravity effects, and a damage definition. Handles collision with geometry and entities, triggering effects and damage on impact. Some projectiles are guided (tracking), some bounce, some explode on contact.
→ *Detailed in [Section 8.2: Projectile System](#82-projectile-system)*

**Weapon System** (`weapons.c`)
The most complex entity system. Manages 10 weapon types with primary/secondary triggers, ammo consumption, reload timing, and firing modes. Supports dual-wielding for pistols and shotguns. Weapons have charging states, heat buildup, and idle animations. Creates projectiles and triggers sound effects on fire.
→ *Detailed in [Section 8.3: Weapon System](#83-weapon-system)*

**Platform System** (`platforms.c`)
Doors, elevators, and crushers. Platforms are polygons that move vertically (floor/ceiling changes). They have states (extending, contracting, idle), speeds, delays, and trigger conditions. Can be activated by players, switches, or other platforms. Some crush entities caught in them.
→ *Detailed in [Section 8.4: Platform System](#84-platform-system)*

**Lighting System** (`lightsource.c`)
Dynamic lighting with multiple light types (normal, strobe, flicker, etc.). Each light has intensity phases that affect nearby polygon shading. Lights are updated every tick and their intensity feeds into the shading table lookup during rendering. Creates atmospheric effects like pulsing lights and flickering torches.
→ *Detailed in [Section 5.4: Lighting and Shading](#54-lighting-and-shading)*

**Sound System** (`game_sound.c`)
3D positioned audio with distance attenuation and stereo panning. Sounds have priorities and limited channels (Mac had 4-8 channels). Calculates volume and pan based on listener position and sound source position. Ambient sounds loop continuously, action sounds play once.
→ *Detailed in [Section 13: Sound System](#13-sound-system)*

**Item System** (`items.c`)
Pickups scattered around levels: ammo, weapons, health, powerups. Items sit in the world as objects and are collected on player contact. Each item type grants specific benefits (ammo counts, weapon unlocks, health points). Some items respawn in multiplayer.
→ *Detailed in [Section 14: Items & Inventory System](#14-items--inventory-system)*

**Effects System** (`effects.c`)
Visual feedback: explosions, sparks, bullet impacts, blood splats. Effects are short-lived objects with animated sprites. They don't interact with physics but provide crucial visual feedback for combat. Each effect has a lifespan and animation sequence.
→ *Detailed in [Section 26: Visual Effects System](#26-visual-effects-system)*

**Media System** (`media.c`)
Liquids that fill polygon volumes: water, lava, sewage, goo. Media has a surface height that can change (tides, draining). Affects player physics (swimming, damage from lava), rendering (tinted view, surface effects), and sound (underwater ambience).
→ *Detailed in [Section 25: Media/Liquid System](#25-medialiquid-system)*

**Shapes System** (`shapes.c`)
Manages texture and sprite collections. Shapes are organized into 32 collections (environment textures, monster sprites, weapon graphics, etc.). Each collection contains multiple bitmaps with optional RLE compression. Color tables and shading tables are built per collection.
→ *Detailed in [Section 19: Shape Animation System](#19-shape-animation-system)*

**HUD System** (`game_window.c`, `interface.c`)
Heads-up display rendering: health bar, oxygen, ammo count, weapon sprite, motion sensor. The HUD draws over the 3D view. The motion sensor is particularly complex, showing nearby entities as blips with distance/direction encoding.
→ *Detailed in [Section 21: HUD Rendering System](#21-hud-rendering-system)*

**Automap System** (`overhead_map.c`)
Top-down map view showing explored areas. Draws polygons as filled shapes, lines as edges, and entities as markers. Only shows areas the player has visited. Can be overlaid on gameplay or shown full-screen.
→ *Detailed in [Section 20: Automap/Overhead Map System](#20-automapoverhead-map-system)*

**Terminal System** (`computer_interface.c`)
In-game computer interfaces for story exposition. Terminals display formatted text, images, and can trigger teleportation or level changes. The terminal format supports multiple pages, text styling, and checkpoint functionality.
→ *Detailed in [Section 28: Computer Terminal System](#28-computer-terminal-system)*

**Network System** (`network.c`)
Peer-to-peer multiplayer for up to 8 players. Uses deterministic lockstep: all players simulate the same world with synchronized inputs. Only action flags are transmitted (tiny packets). If simulations diverge, the game detects desync via checksums. Supports game types like deathmatch, cooperative, king of the hill.
→ *Detailed in [Section 9: Networking Architecture](#9-networking-architecture)*

**File System** (`wad.c`, `game_wad.c`)
Marathon's custom file format for maps and saved games. WAD files contain tagged data chunks (polygons, objects, textures references, etc.). Not related to Doom WADs despite the name. Shapes and sounds use a separate binary format (data fork files).
→ *Detailed in [Section 10: File Formats](#10-file-formats)*

### Coordinate System and Units

Marathon uses a right-handed coordinate system with fixed-point values for all positions, distances, and angles.

#### World Coordinate System

```
                        Top-Down View (X-Y Plane)

                              +Y (North)
                                ↑
                                │
                                │
                                │
                 ───────────────┼───────────────→ +X (East)
                                │
                                │
                                │
                                ↓
                              -Y (South)

                        +Z is UP (out of page)
                        -Z is DOWN (into page)

3D View:
                   +Z (Up)
                     ↑
                     │    +Y (North)
                     │   ╱
                     │  ╱
                     │ ╱
                     │╱
     ────────────────┼────────────────→ +X (East)
                    ╱│
                   ╱ │
                  ╱  │
                 ╱   │
              -Y     ↓
                   -Z (Down)
```

#### Unit System

Marathon uses two fixed-point number formats for different precision needs:

**World Units** (10 fractional bits):
```c
typedef short world_distance;  // 16-bit signed, 10 fractional bits

#define WORLD_FRACTIONAL_BITS 10
#define WORLD_ONE 1024              // 1.0 in world units
#define WORLD_ONE_HALF 512          // 0.5 in world units
#define WORLD_ONE_FOURTH 256        // 0.25 in world units

// Conversions
#define INTEGER_TO_WORLD(s)    (((world_distance)(s))<<WORLD_FRACTIONAL_BITS)
#define WORLD_INTEGERAL_PART(d) ((d)>>WORLD_FRACTIONAL_BITS)
#define WORLD_FRACTIONAL_PART(d) ((d)&((world_distance)(WORLD_ONE-1)))
```

**Fixed-Point** (16 fractional bits, for higher precision):
```c
typedef long fixed;  // 32-bit signed, 16 fractional bits

#define FIXED_FRACTIONAL_BITS 16
#define FIXED_ONE 65536             // 1.0 in fixed-point

// Conversions
#define INTEGER_TO_FIXED(s)    (((fixed)(s))<<FIXED_FRACTIONAL_BITS)
#define FIXED_TO_INTEGER(f)    ((f)>>FIXED_FRACTIONAL_BITS)

// Between formats
#define WORLD_TO_FIXED(w) (((fixed)(w))<<6)   // Multiply by 64
#define FIXED_TO_WORLD(f) ((world_distance)((f)>>6))  // Divide by 64
```

**Scale Reference**:
```
WORLD_ONE (1024 units) ≈ 1 game "world unit"

Approximate real-world equivalents:
┌────────────────────────────┬──────────────────┐
│ Game Measurement           │ World Units      │
├────────────────────────────┼──────────────────┤
│ Player height              │ ~819 (0.8 WU)    │
│ Player radius              │ ~256 (0.25 WU)   │
│ Door width                 │ ~1024-2048       │
│ Typical room               │ ~4096-8192       │
│ Step-up height (max)       │ ~341 (1/3 WU)    │
│ Wall separation distance   │ ~256 (1/4 WU)    │
└────────────────────────────┴──────────────────┘
```

#### Angle System

```c
typedef short angle;  // 16-bit, but only 9 bits used

#define ANGULAR_BITS 9
#define NUMBER_OF_ANGLES 512        // Full circle = 512 units
#define FULL_CIRCLE 512
#define HALF_CIRCLE 256             // 180°
#define QUARTER_CIRCLE 128          // 90°
#define EIGHTH_CIRCLE 64            // 45°
#define SIXTEENTH_CIRCLE 32         // 22.5°

#define NORMALIZE_ANGLE(t) ((t)&(angle)(NUMBER_OF_ANGLES-1))
```

**Angle Visualization**:
```
                  128 (North/+Y)
                         ↑
                         │
           192 ┌─────────┼─────────┐ 64
           (NW)│         │         │(NE)
               │         │         │
               │         │         │
  256 ─────────┼────⊕────┼─────────→ 0 (East/+X)
  (West/-X)    │         │         │
               │  angles │         │
               │ increase│         │
               │   CCW   │         │
           320 └─────────┼─────────┘ 448
           (SW)          │         (SE)
                         ↓
                    384 (South/-Y)

Note: Angles increase COUNTER-CLOCKWISE (mathematical convention)
      0 = facing +X direction (East)
      128 = facing +Y direction (North)
      256 = facing -X direction (West)
      384 = facing -Y direction (South)
```

**Angle Conversion Table**:
| Marathon Angle | Degrees | Radians | Direction |
|----------------|---------|---------|-----------|
| 0 | 0° | 0 | East (+X) |
| 64 | 45° | π/4 | Northeast |
| 128 | 90° | π/2 | North (+Y) |
| 192 | 135° | 3π/4 | Northwest |
| 256 | 180° | π | West (-X) |
| 320 | 225° | 5π/4 | Southwest |
| 384 | 270° | 3π/2 | South (-Y) |
| 448 | 315° | 7π/4 | Southeast |
| 512 | 360° | 2π | East (wraps) |

**Quick Conversion**: `degrees = (angle × 360) / 512` or `angle = (degrees × 512) / 360`

#### Point and Vector Structures

```c
// 2D point in world space (floor plan)
struct world_point2d {
    world_distance x, y;  // 4 bytes total
};

// 3D point in world space
struct world_point3d {
    world_distance x, y, z;  // 6 bytes total
};

// High-precision 3D point (for physics)
struct fixed_point3d {
    fixed x, y, z;  // 12 bytes total
};

// 2D direction vector
struct world_vector2d {
    world_distance i, j;  // Components along X, Y axes
};

// 3D direction vector
struct world_vector3d {
    world_distance i, j, k;  // Components along X, Y, Z axes
};

// Full location with orientation
struct world_location3d {
    world_point3d point;      // Position
    short polygon_index;      // Which polygon we're in
    angle yaw, pitch;         // Facing direction
    world_vector3d velocity;  // Movement vector
};
```

#### Trigonometry

Marathon uses precomputed lookup tables for all trigonometric functions:

```c
#define TRIG_SHIFT 10
#define TRIG_MAGNITUDE 1024  // Trig results scaled by this

extern short sine_table[NUMBER_OF_ANGLES];    // 512 entries
extern short cosine_table[NUMBER_OF_ANGLES];  // 512 entries

// Usage: multiply result by distance, then shift right by TRIG_SHIFT
// Example: x = distance * cosine_table[angle] >> TRIG_SHIFT
```

**Trig Table Values**:
```
Angle    sin(angle)    cos(angle)    Notes
─────────────────────────────────────────────
0        0             1024          East (+X)
64       724           724           NE (45°)
128      1024          0             North (+Y)
192      724           -724          NW (135°)
256      0             -1024         West (-X)
320      -724          -724          SW (225°)
384      -1024         0             South (-Y)
448      -724          724           SE (315°)
```

#### Transform Operations

**Translation** (move point by distance in direction):
```c
world_point2d *translate_point2d(
    world_point2d *point,     // Point to modify
    world_distance distance,  // How far to move
    angle theta               // Direction to move
);

// Implementation:
point->x += (distance * cosine_table[theta]) >> TRIG_SHIFT;
point->y += (distance * sine_table[theta]) >> TRIG_SHIFT;
```

**Rotation** (rotate point around origin):
```c
world_point2d *rotate_point2d(
    world_point2d *point,     // Point to rotate
    world_point2d *origin,    // Center of rotation
    angle theta               // Rotation amount (CCW positive)
);
```

**Transform** (translate to view space - used in rendering):
```c
world_point2d *transform_point2d(
    world_point2d *point,     // World-space point
    world_point2d *origin,    // Camera position
    angle theta               // Camera facing angle
);

// Result: point relative to camera, rotated so camera faces +X
```

#### View Space vs World Space

```
World Space:                    View Space (after transform):

    +Y (North)                      +Y (Left of camera)
      ↑                               ↑
      │   Player facing               │
      │   this way →                  │   Camera always
      │                               │   faces this way →
      ├───────→ +X (East)             ├───────→ +X (Forward)
      │                               │
      │                               │
      ↓                               ↓
    -Y (South)                      -Y (Right of camera)

transform_point2d() converts from world to view space:
1. Subtract camera position (translate origin)
2. Rotate so camera's facing direction becomes +X
```

#### Screen Space

After perspective projection, 3D points become 2D screen coordinates:

```
Screen Coordinate System:

(0,0) ────────────────────────→ +X (screen_width-1, 0)
  │
  │     ┌─────────────────┐
  │     │                 │
  │     │   Game View     │
  │     │                 │
  │     │    (640×480)    │
  │     │                 │
  │     └─────────────────┘
  │
  ↓
(0, screen_height-1)

Screen center: (half_width, half_height)
  - half_width = 320 (for 640 wide)
  - half_height = 240 (for 480 tall)

Projection formulas:
  screen_x = half_width + (view_y × world_to_screen_x) / view_x
  screen_y = half_height - (view_z × world_to_screen_y) / view_x + pitch_offset

Note: Y is inverted (screen Y increases downward, world Z increases upward)
```

---

## 4. World Representation

> **🔧 For Porting:** `map.c`, `map.h`, and `map_constructors.c` are fully portable. All data structures use platform-independent types (`short`, `long`, `word`). Just ensure `word` is defined as `unsigned short` and handle byte swapping when loading from files.

### Polygon-Based Geometry

Marathon uses explicit polygon connectivity, not hierarchical BSP trees.

#### Core Data Structures

**Points (Vertices)**

The fundamental 2D point:
```c
struct world_point2d {  // 4 bytes
    world_distance x, y;  // Each is a short (16-bit)
};
```

Marathon stores vertices as **endpoints** [map.h:378] with additional metadata (16 bytes):
```c
struct endpoint_data {
    word flags;
    world_distance highest_adjacent_floor_height;
    world_distance lowest_adjacent_ceiling_height;
    world_point2d vertex;      // The actual 2D position
    world_point2d transformed; // View-space coordinates (cached)
    short supporting_polygon_index;
};
```

**Lines (Edges)** [map.h:416] (32 bytes)
```c
struct line_data {
    short endpoint_indexes[2];           // Point indices
    word flags;                          // Solid, transparent, etc.
    short clockwise_polygon_owner;        
    short counterclockwise_polygon_owner;
    short clockwise_polygon_side_index;
    short counterclockwise_polygon_side_index;
};
```

**Line Flags** (stored in `line_data.flags`):

| Flag | Bit | Description |
|------|-----|-------------|
| `SOLID_LINE_BIT` | 0x4000 | Blocks movement (impassable wall) |
| `TRANSPARENT_LINE_BIT` | 0x2000 | Can see through to adjacent polygon |
| `LANDSCAPE_LINE_BIT` | 0x1000 | Uses landscape (sky) texture |
| `ELEVATION_LINE_BIT` | 0x800 | Has elevation change |
| `VARIABLE_ELEVATION_LINE_BIT` | 0x400 | Elevation varies (platform) |
| `LINE_HAS_TRANSPARENT_SIDE_BIT` | 0x200 | Has glass/window texture |

**Usage Macros**: `LINE_IS_SOLID(l)`, `LINE_IS_TRANSPARENT(l)`, `LINE_IS_LANDSCAPED(l)`, etc.

**Sides (Wall Surfaces) - 64 bytes**
```c
struct side_data {
    word flags;
    short type;  // _full_side, _high_side, _low_side, etc.
    
    struct side_texture_definition {
        world_distance x0, y0;
        shape_descriptor texture;
    } primary_texture, secondary_texture, transparent_texture;
    
    short control_panel_type;      // Switch, terminal, etc.
    short primary_lightsource_index;
    fixed ambient_delta;
};
```

**Polygons (Rooms)** [map.h:571] (128 bytes)
```c
struct polygon_data {
    byte type;                     // Normal, platform, teleporter, etc.
    word flags;
    short vertex_count;            // Max 8 vertices
    short endpoint_indexes[8];
    short line_indexes[8];
    short adjacent_polygon_indexes[8];
    
    world_distance floor_height, ceiling_height;
    short floor_texture, ceiling_texture;
    short floor_lightsource_index, ceiling_lightsource_index;
    
    long area;                     // Precomputed
    short first_object;            // Object linked list
    short media_index;             // Liquid type
};
```

**Polygon Types**:

| Type | Value | Description | .permutation Use |
|------|-------|-------------|------------------|
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
| `_polygon_is_zone_border` | 11 | Zone boundary | Unused |
| `_polygon_is_goal` | 12 | Level exit | Unused |
| `_polygon_is_visible_monster_trigger` | 13 | Activates monsters (sight) | Unused |
| `_polygon_is_invisible_monster_trigger` | 14 | Activates monsters (entry) | Unused |
| `_polygon_is_dual_monster_trigger` | 15 | Both trigger types | Unused |
| `_polygon_is_item_trigger` | 16 | Activates items in zone | Unused |
| `_polygon_is_automatic_exit` | 18 | Auto-exit on success | Unused |

### Connectivity Graph

Polygons connect through lines:

```
Polygon A ←──Line 1──→ Polygon B
    ↑                      ↓
  Line 4                Line 2
    ↑                      ↓
Polygon D ←──Line 3──→ Polygon C
```

**Detailed Connectivity Visualization**:

```
How Marathon's World Geometry Connects:

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

**Clockwise/Counterclockwise Owner System**:

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

        Code usage:
        if (current_polygon == line->clockwise_polygon_owner) {
            // We're in Polygon A, so across the line is...
            next_polygon = line->counterclockwise_polygon_owner; // Polygon B
        } else {
            // We're in Polygon B, so across the line is...
            next_polygon = line->clockwise_polygon_owner; // Polygon A
        }
```

**Complete Example: 4-Polygon Room**:

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
         e4 ─────L6───── e5

Data in memory:

Endpoints array (6 total):
  [0]: {x=512, y=512, ...}     // Center-top
  [1]: {x=1024, y=512, ...}    // Right-top
  [2]: {x=1024, y=1024, ...}   // Right-bottom
  [3]: {x=512, y=1024, ...}    // Center-bottom
  [4]: {x=256, y=1280, ...}    // Left-bottom
  [5]: {x=768, y=1280, ...}    // Right-far-bottom

Lines array (7 total):
  [0]: {endpoints={0,1}, cw_owner=0, ccw_owner=1, ...}  // Top edge
  [1]: {endpoints={1,2}, cw_owner=0, ccw_owner=2, ...}  // Right edge
  [2]: {endpoints={2,3}, cw_owner=0, ccw_owner=-1, ...} // Bottom edge (solid wall!)
  [3]: {endpoints={3,0}, cw_owner=0, ccw_owner=3, ...}  // Left edge
  [4]: {endpoints={3,4}, cw_owner=4, ccw_owner=-1, ...} // Extension
  [5]: {endpoints={3,5}, cw_owner=5, ccw_owner=-1, ...} // Extension
  [6]: {endpoints={4,5}, cw_owner=4, ccw_owner=5, ...}  // Far edge

Polygons array (6 total):
  [0]: {vertex_count=4, endpoints={0,1,2,3}, lines={0,1,2,3},
        adjacent={1,2,-1,3}, floor=0, ceiling=1024, ...}  // Main room
  [1]: {vertex_count=4, endpoints={...}, adjacent={0,...}, ...}  // North room
  [2]: {vertex_count=4, endpoints={...}, adjacent={0,...}, ...}  // East room
  [3]: {vertex_count=4, endpoints={...}, adjacent={0,...}, ...}  // West room
  [4]: {vertex_count=3, endpoints={3,4,5}, adjacent={-1,5,-1}, ...} // South-left
  [5]: {vertex_count=3, endpoints={3,5,2}, adjacent={-1,4,-1}, ...} // South-right

Traversal example - player walks from Polygon 0 to Polygon 2:
  1. Player in Polygon 0 crosses Line 1 (right edge)
  2. Line 1 has: clockwise_owner=0, counterclockwise_owner=2
  3. Since we came from polygon 0 (clockwise), we enter polygon 2 (counterclockwise)
  4. Now player is in Polygon 2 (East room)

This connectivity allows O(1) traversal - no searching needed!
```

**Adjacent Polygon Lookup** - O(1):
```c
short find_adjacent_polygon(short polygon_index, short line_index)
{
    struct line_data *line = get_line_data(line_index);
    return (polygon_index == line->clockwise_polygon_owner) ?
           line->counterclockwise_polygon_owner :
           line->clockwise_polygon_owner;
}
```

### Platforms (Dynamic Geometry)

**Platform Structure**:
```c
struct platform_data {
    short type;                    // Door, elevator, etc.
    short speed, delay;
    world_distance minimum_floor_height, maximum_floor_height;
    world_distance minimum_ceiling_height, maximum_ceiling_height;
    
    word dynamic_flags;            // Active, extending, contracting
    world_distance floor_height, ceiling_height;
    short polygon_index;
};
```

**Platform Types**:
- `_platform_is_spht_door` - Fast door
- `_platform_is_heavy_spht_door` - Slow, heavy door
- `_platform_is_spht_platform` - Standard elevator
- `_platform_is_pfhor_door` - Alien door
- Split doors, locked doors, etc.

**Movement Speeds**:
```c
_very_slow_platform = WORLD_ONE/(4*30)    // ~8.5 units/second
_slow_platform = WORLD_ONE/(2*30)         // ~17 units/second  
_fast_platform = 2 * _slow_platform       // ~34 units/second
```

### Media (Liquids)

**Media Structure**:
```c
struct media_data {
    short type;                    // Water, lava, goo, sewage, jjaro
    short light_index;             // For animated height!
    angle current_direction;       // Flow direction
    world_distance current_magnitude;
    world_distance low, high;      // Height range
    world_distance height;         // Current level
};
```

**Clever Height System**:
Media height calculated from light intensity:
```c
height = low + FIXED_INTEGERAL_PART((high - low) * get_light_intensity(light_index))
```

This allows water levels to rise/fall by simply animating a light source!

### Lighting System

**Light Types**:
- `_normal_light` - On/off
- `_strobe_light` - Blinking
- `_media_light` - Tied to liquid height

**Lighting Functions**:
- `_constant_lighting_function` - Instant
- `_linear_lighting_function` - Linear fade
- `_smooth_lighting_function` - Sine-based
- `_flicker_lighting_function` - Random

**Light State Machine**:
```c
enum {
    _light_becoming_active,
    _light_primary_active,
    _light_secondary_active,
    _light_becoming_inactive,
    _light_primary_inactive,
    _light_secondary_inactive
};
```

Each state has independent transition parameters (period, intensity, function).

### Object Placement and Spawning System

Marathon has a sophisticated system for placing objects in maps and dynamically respawning them during gameplay.

**Initial Object Placement**:
```c
#define MAXIMUM_SAVED_OBJECTS 384

enum {  // Map object types
    _saved_monster,      // .index is monster type
    _saved_object,       // .index is scenery type
    _saved_item,         // .index is item type
    _saved_player,       // .index is team bitfield
    _saved_goal,         // .index is goal number
    _saved_sound_source  // .index is source type, .facing is volume
};

enum {  // Map object flags
    _map_object_is_invisible = 0x0001,           // Initially invisible
    _map_object_hanging_from_ceiling = 0x0002,   // For position calculation
    _map_object_is_blind = 0x0004,               // Monster cannot activate by sight
    _map_object_is_deaf = 0x0008,                // Monster cannot activate by sound
    _map_object_floats = 0x0010,                 // Floating object
    _map_object_is_network_only = 0x0020         // Only appears in multiplayer
};

struct map_object {
    short type;
    short index;
    short facing;
    short polygon_index;
    world_point3d location;  // .z is delta from polygon floor
    word flags;
};
```

**Dynamic Respawning System**:
```c
struct object_frequency_definition {
    word flags;
    short initial_count;    // How many appear at map start
    short minimum_count;    // Minimum maintained (if replenish enabled)
    short maximum_count;    // Maximum cap
    short random_count;     // Max random appearances
    word random_chance;     // Spawn probability [0, 65535]
};
```

**Placement Flags**:
- `_reappears_in_random_location` - Objects respawn at random positions when destroyed

**Respawn Algorithm** (from `recreate_objects()`):
1. Check current object count vs minimum_count
2. If below minimum and `_monsters_replenish` game option enabled:
   - Select random polygon weighted by polygon area
   - Find random point within selected polygon
   - Verify visibility requirements (some items must be visible to player)
   - Verify object type is valid for that polygon
   - Spawn new object instance

**Difficulty Modulation**:
- **Wuss/Easy difficulty**: Randomly drop 1/4 to 1/8 of monsters at map start
- **Major monsters**: Demoted to minor variants on lower difficulties
- Affects spawn density, not individual monster abilities

This system ensures:
- Maps never become completely empty of monsters/items
- Difficulty scales appropriately
- Players always have resources available
- Multiplayer games maintain balance

---

## 5. Rendering System

> **For frame-by-frame rendering flow**: See Section 32 (Life of a Frame) for complete pipeline diagrams, clipping window accumulation, and texture mapping function reference.

> **🔧 For Porting:** The core renderer (`render.c`, 3,879 lines) is 99% portable C! Only changes needed:
> - Replace `world_pixels` GWorld with your framebuffer (32-bit ARGB)
> - Modify `scottish_textures.c` to write 32-bit pixels instead of 8-bit palette indices
> - Remove/stub assembly texture mappers (`.a` files)—C fallbacks exist
> - See `screen.c` for framebuffer setup that needs platform replacement

### The Rendering Pipeline

**Main Entry Point: `render_view()`** (render.c:3879 lines)

```
1. update_view_data()              Update camera transform
2. build_render_tree()             Portal visibility culling
3. sort_render_tree()              Depth-order polygons
4. build_render_object_list()      Collect sprites
5. render_tree()                   Draw polygons back-to-front
   ├─ Render ceilings (if above)
   ├─ Render walls/sides
   ├─ Render interior objects
   └─ Render floors (if below)
6. render_viewer_sprite_layer()    Draw weapon/HUD
```

### Portal-Based Visibility Culling

**Node Tree Structure**:
```c
struct node_data {
    word flags;
    short polygon_index;
    
    short clipping_endpoint_count;
    short clipping_endpoints[MAXIMUM_CLIPPING_ENDPOINTS_PER_NODE];
    short clipping_line_count;
    short clipping_lines[MAXIMUM_CLIPPING_LINES_PER_NODE];
    
    struct node_data *parent;
    struct node_data *children, *siblings;
};
```

**Build Algorithm** (`build_render_tree()`):

1. **Initialize root node** with viewer's polygon
2. **Cast rays at screen edges** (render.c:714-715):
   - Left edge with `_counterclockwise_bias` - cross lines counterclockwise from endpoint
   - Right edge with `_clockwise_bias` - cross lines clockwise from endpoint
   - Bias determines which polygon to enter when ray hits exactly on a vertex
3. **Queue polygons** visible through portals
4. **For each queued polygon**:
   - Cast rays through all vertices
   - Find which adjacent polygons are visible
   - Recursively add to tree
5. **Accumulate clipping data** from portal crossings

**Ray Casting** (`cast_render_ray()`):

```c
// Traces ray through polygon graph
// Returns which polygon ray exits through
word next_polygon_along_line(
    short *polygon_index,
    world_point2d *origin,
    world_vector2d *vector,
    ...)
{
    // Test ray against each polygon edge
    for (each edge) {
        // Cross product test
        long cross = (ray.x - e0->x) * (e1->y - e0->y) -
                     (ray.y - e0->y) * (e1->x - e0->x);

        if (cross > 0) {
            // Ray exits through this edge
            *polygon_index = find_adjacent_polygon(*polygon_index, line_index);
            return clip_flags;
        }
    }
}
```

#### Ray Casting Visualization

**Cross-Product Test for Line Intersection**:

```
                    e1 (endpoint 1)
                     *
                    /|
                   / |
                  /  |    ← edge vector (e1 - e0)
                 /   |
                /    |
               /     |
              /      |
         e0  *-------+-------* ray.x, ray.y
         (endpoint 0)

Test: cross = (ray.x - e0.x) × (e1.y - e0.y) - (ray.y - e0.y) × (e1.x - e0.x)

If cross > 0: Ray is to the RIGHT of edge → exits through this edge
If cross < 0: Ray is to the LEFT of edge → inside polygon
If cross = 0: Ray is ON the edge
```

**Ray Traversal Through Polygon Graph**:

```
         Polygon A              Polygon B              Polygon C
    ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
    │              │       │              │       │              │
    │   Player     │       │              │       │   Target     │
    │     @────────┼───────┼──────────────┼───────┼────>*        │
    │              │       │              │       │              │
    └──────────────┘       └──────────────┘       └──────────────┘
         Line 1                 Line 2                 Line 3

Algorithm:
1. Start in Polygon A
2. Test ray against edges (0, 1, 2, 3)
3. Cross > 0 at Line 1 → move to Polygon B
4. Test ray against Polygon B edges
5. Cross > 0 at Line 2 → move to Polygon C
6. Continue until ray exits map or hits solid wall
```

**Portal Visibility Test**:

```
                        Screen
                     ╔═════════╗
          Left Ray  ╱           ╲  Right Ray
                   ╱             ╲
                  ╱               ╲
                 ╱    View Cone    ╲
                ╱                   ╲
               ╱         @          ╲
                    (Player)

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

**Clipping Windows**:
```c
struct clipping_window_data {
    world_vector2d left, right, top, bottom;  // Clipping vectors
    short x0, x1, y0, y1;                     // Screen-space bounds
    struct clipping_window_data *next_window;
};
```

Portal crossings accumulate clipping bounds for each visible polygon.

#### Portal Node Tree Building

**Tree Structure** (Recursive Portal Traversal):

```
                         Node 0
                    (Player's Polygon)
                    [x0=0, x1=639]
                          |
          ┌───────────────┼───────────────┐
          │               │               │
       Node 1          Node 2          Node 3
    (Left Portal)   (Center Portal)  (Right Portal)
    [x0=0, x1=200]  [x0=200, x1=450] [x0=450, x1=639]
          │               │               │
      ┌───┴───┐       ┌───┴───┐          │
   Node 1.1 Node 1.2  ...              Node 3.1
   [x0=0,   [x0=100,                   [x0=450,
    x1=100]  x1=200]                    x1=639]

Each node stores:
- Polygon index to render
- Clipping window (x0, x1, y0, y1)
- Children (visible through this polygon's portals)
```

**Build Process** (Per-Polygon Expansion):

```
Step 1: Initialize root node with player's polygon

         @  ← Player in Polygon 0
    ┌────────────┐
    │ Polygon 0  │  → Root Node (x: 0-639, y: 0-479)
    └────────────┘

Step 2: Cast rays through all visible portals from Polygon 0

         @
    ┌────┬────┬──┐
    │ P0 │ P1 │P2│  → Cast ray to each portal edge
    └────┴────┴──┘
         ↓    ↓
      Portal  Portal
      to P1   to P2

Step 3: For each portal, calculate clipping window

    Portal to P1:
    - Left edge: ray through leftmost vertex  → screen_x = 100
    - Right edge: ray through rightmost vertex → screen_x = 300
    - Create child node [x0=100, x1=300]

Step 4: Recursively process child nodes
    - Start with child's clipping window as constraint
    - Find portals visible within that window
    - Add grandchildren nodes
```

#### Scanline Rendering Order

**Depth-First Tree Traversal** (Back-to-Front):

```
Render Tree:           Render Order (Depth-First):

    Node A                 1. Render subtree B
    [Poly 0]                  ├─ Render B's floor
   ╱       ╲                  ├─ Render B's walls
Node B    Node C              └─ Render B's objects
[Poly 1]  [Poly 3]         2. Render subtree C
   │                          ├─ Render subtree D
Node D                        │  └─ Process Poly 4
[Poly 4]                      ├─ Render C's floor
                              ├─ Render C's walls
                              └─ Render C's objects
                           3. Render Node A
                              ├─ Render A's floor
                              ├─ Render A's walls
                              └─ Render A's objects

Result: Painters algorithm (far to near)
```

**Per-Scanline Processing**:

```
For each polygon, render scanline by scanline:

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

### Perspective Projection

**View Setup** (`initialize_view_data()`):

```c
// Field of view calculation
double half_cone = field_of_view * (2π/360) / 2;
double adjusted_half_cone = asin(screen_width * sin(half_cone) / standard_screen_width);

// World-to-screen scale factor
double world_to_screen = (half_screen_width) / tan(adjusted_half_cone);
view->world_to_screen_x = (short)(world_to_screen / horizontal_scale);
view->world_to_screen_y = (short)(world_to_screen / vertical_scale);
```

**Field of View**:
- Normal: 80 degrees
- Extravision: 130 degrees

**Projection Math**:
```c
// Transform to viewer coordinates
transform_point2d(&endpoint->transformed, &view->origin, view->yaw);

// Project to screen
screen_x = half_width +
           (transformed.y * world_to_screen_x) / transformed.x;
screen_y = half_height -
           (transformed.z * world_to_screen_y) / transformed.x + dtanpitch;
```

Where `dtanpitch = world_to_screen_y * tan(pitch)` for view pitch compensation.

#### Pitch Calculation (Looking Up/Down)

**How Vertical View Angle Works**:

```
Looking level (pitch = 0):
                                Screen
    World Z                  ┌─────────┐
       ↑                     │    ·    │ ← half_height (center)
       │                     │         │
   ────┼────  Z = eye_level  │    ·    │
       │                     │         │
       │                     └─────────┘
       └─→ View direction    dtanpitch = 0

Looking up (pitch > 0):
                                Screen
    World Z                  ┌─────────┐
       ↑  ╱                  │         │
       │ ╱ pitch             │    ·    │ ← Shifted down by dtanpitch
   ────┼────  Z = eye_level  │         │
       │                     │    ·    │
       └─→ View direction    └─────────┘
                             dtanpitch > 0 (positive shift)

Looking down (pitch < 0):
                                Screen
    World Z                  ┌─────────┐
       │                     │    ·    │
   ────┼────  Z = eye_level  │         │
       │ ╲                   │    ·    │ ← Shifted up by dtanpitch
       ↓  ╲ pitch            └─────────┘
          └─→ View direction dtanpitch < 0 (negative shift)
```

**dtanpitch Calculation**:

```c
// Pitch is stored as a fixed-point angle
// tan(pitch) shifts all vertical coordinates

dtanpitch = world_to_screen_y * tan(pitch)

Example with pitch = +15 degrees (looking up):
- world_to_screen_y = 300 (example value)
- tan(15°) ≈ 0.268
- dtanpitch = 300 * 0.268 = 80 pixels

Result: All points shifted down 80 pixels on screen
        → Looking up shows more ceiling
```

**Complete Screen Projection**:

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

### Texture Mapping

Marathon uses **two separate texture mappers**:

#### 1. Horizontal Surfaces (Floors/Ceilings)

**Function**: `texture_horizontal_polygon()` [scottish_textures.c:277]

**Algorithm**:
1. **Build edge tables** - Bresenham's algorithm for left/right x-coordinates per scanline
2. **Precalculate texture coordinates** per scanline:
   ```c
   source_x = (dhcosine - screen_x*hsine)/screen_y + (origin.x << TRIG_SHIFT);
   source_dx = -hsine/screen_y;
   source_y = (screen_x*hcosine + dhsine)/screen_y + (origin.y << TRIG_SHIFT);
   source_dy = hcosine/screen_y;
   ```
3. **Rasterize** - Assembly-optimized inner loop

**Data Structure**:
```c
struct _horizontal_polygon_line_data {
    unsigned long source_x, source_y;     // Texture coordinates (fixed-point)
    unsigned long source_dx, source_dy;   // Deltas per pixel
    void *shading_table;                  // Precomputed lighting lookup
};
```

Uses **affine mapping** (fast but approximate) - acceptable for horizontal surfaces.

**Horizontal Texture Mapping Visualization**:

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

**Affine vs Perspective-Correct Comparison**:

```
Affine mapping (what would happen on a wall):

        Texture (8×8 grid):              Wall in 3D:              Screen result:
        ┌─┬─┬─┬─┬─┬─┬─┬─┐                   far                  ┌─┬─┬─┬─────────┐
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │                   ├─┼─┼─┼─────────┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │  Receding         ├─┼─┼─┼─────────┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │  into             ├─┼─┼─┼─────────┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │  distance         ├─┼─┼─┼─────────┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │                   ├─┼─┼─┼─────────┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │                   ├─┼─┼─┼─────────┤
        └─┴─┴─┴─┴─┴─┴─┴─┘                  near                 └─┴─┴─┴─────────┘
                                                                 INCORRECT!
                                                                 (stretched at bottom)

Perspective-correct mapping (Marathon's wall approach):

        Texture (8×8 grid):              Wall in 3D:              Screen result:
        ┌─┬─┬─┬─┬─┬─┬─┬─┐                   far                  ┌─┬─┬─┬─┬─┬─┬─┬─┐
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │                   ├─┼─┼─┼─┼─┼─┼─┼─┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │  Receding         ├─┼─┼─┼─┼─┼─┼─┼─┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │  into             ├─┼─┼─┼─┼─┼─┼─┼─┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │  distance         ├─┼─┼─┼─┼─┼─┼─┼─┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │                   ├─┼─┼─┼─┼─┼─┼─┼─┤
        ├─┼─┼─┼─┼─┼─┼─┼─┤                    │                   ├─┼─┼─┼─┼─┼─┼─┼─┤
        └─┴─┴─┴─┴─┴─┴─┴─┘                  near                 └─┴─┴─┴─┴─┴─┴─┴─┘
                                                                 CORRECT!
                                                                 (even spacing)
```

#### 2. Vertical Surfaces (Walls)

**Function**: `texture_vertical_polygon()` [scottish_textures.c:476]

**Algorithm**:
1. **Build column tables** - y-coordinates for top/bottom per screen column
2. **Precalculate per column**:
   ```c
   // Texture x (horizontal position on wall)
   x0 = ((tx_numerator << VERTICAL_TEXTURE_WIDTH_BITS) / tx_denominator) & mask;
   
   // Texture y (vertical position)
   ty = INTEGER_TO_FIXED(ty_numerator) / ty_denominator;
   ty_delta = -INTEGER_TO_FIXED(world_x) / (ty_denominator >> 8);
   ```
3. **Rasterize column-wise** - Perspective-correct

Uses **perspective-correct mapping** for walls to avoid distortion.

**Vertical Texture Mapping Visualization**:

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

**Column-Based Wall Rendering**:

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

**Perspective-Correct Calculation Detail**:

```
Why world_x matters for perspective:

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

          Closer walls (small world_x) → smaller ty_delta → slower texture movement
          Farther walls (large world_x) → larger ty_delta → faster texture movement

          Result: Perspective-correct texture mapping!
```

**Complete Polygon Filling Process**:

```
From 3D World Polygon to Screen Pixels - Complete Pipeline

Step 1: WORLD SPACE POLYGON
        3D polygon in game world (floor example):

        World coordinates (WORLD_ONE = 1024 units):
                        v0 (x=2048, y=1024, z=0)
                       /  \
                      /    \
                     /      \
    v3 (x=1024,    /________\    v1 (x=3072,
        y=2048,               \       y=1024,
        z=0)                   \      z=0)
                                \
                                 \
                          v2 (x=2048, y=2048, z=0)

Step 2: TRANSFORM TO VIEW SPACE
        Apply camera transform (rotation + translation):

        view_x = (world.x - camera.x) × cos(yaw) + (world.y - camera.y) × sin(yaw)
        view_y = -(world.x - camera.x) × sin(yaw) + (world.y - camera.y) × cos(yaw)
        view_z = world.z - camera.z

Step 3: PERSPECTIVE PROJECTION
        Project to screen coordinates:

        screen_x = half_width + (view_y × world_to_screen_x) / view_x
        screen_y = half_height - (view_z × world_to_screen_y) / view_x + dtanpitch

        Projected vertices on screen:
                        v0 (x=320, y=150)
                       /  \
                      /    \
                     /      \
                    /        \
                   /          \
        v3 (x=200, /____________\ v1 (x=440, y=150)
            y=300)               \
                                  \
                           v2 (x=320, y=300)

Step 4: EDGE TABLE BUILDING (Horizontal surfaces use this)
        Use Bresenham's algorithm to find left/right edges per scanline:

        Scanline | Left Edge | Right Edge | Status
        ─────────┼───────────┼────────────┼────────
        150      | 320       | 320        | Single pixel (top vertex)
        151      | 319       | 321        | Expanding
        152      | 318       | 322        |
        ...      | ...       | ...        | Trapezoid
        200      | 240       | 400        | Widest part
        ...      | ...       | ...        | Contracting
        299      | 201       | 439        |
        300      | 200       | 440        | Bottom edge

        OR for Vertical surfaces:
        Build column table (top/bottom y per x column)

Step 5: TEXTURE COORDINATE PRECALCULATION
        For EACH scanline (horizontal) or EACH column (vertical):

        Horizontal (floor/ceiling):
          source_x, source_y = starting texture coordinates
          source_dx, source_dy = increment per pixel
          shading_table = lookup based on depth

        Vertical (walls):
          texture_column_index = which column of texture
          texture_y = starting v-coordinate
          texture_dy = increment per pixel (perspective-corrected)
          shading_table = lookup based on world_x distance

Step 6: RASTERIZATION (Inner Loop)
        The actual pixel writing - this runs MILLIONS of times per frame!

        HORIZONTAL SURFACE (scanline y=200, left_x=240, right_x=400):
          while (x < right_x) {
            // Fixed-point texture coordinate (16.16 format)
            u = source_x >> 16;          // Integer part = 10
            v = source_y >> 16;          // Integer part = 15

            // Fetch texture pixel (8-bit palette index)
            texture_pixel = texture[v][u];  // e.g., pixel value = 42

            // Apply lighting via shading table
            shaded_pixel = shading_table[texture_pixel];  // Darkened version

            // Write to framebuffer
            screen[y × screen_width + x] = shaded_pixel;

            // Advance texture coordinates
            source_x += source_dx;  // Move across texture
            source_y += source_dy;
            x++;
          }

        VERTICAL SURFACE (column x=300, top_y=120, bottom_y=280):
          while (y < bottom_y) {
            // Fixed-point v-coordinate
            v = (texture_y >> VERTICAL_TEXTURE_FREE_BITS) & texture_mask;

            // Fetch from pre-selected texture column
            texture_pixel = texture_column[v];

            // Apply lighting
            shaded_pixel = shading_table[texture_pixel];

            // Write to framebuffer
            screen[y × screen_width + x] = shaded_pixel;

            // Advance with perspective-correct delta
            texture_y += texture_dy;
            y++;
          }

Step 7: FINAL FRAMEBUFFER
        After all polygons rendered (back-to-front):

        Screen buffer (640×480 pixels):
        ┌─────────────────────────────────────────┐
        │                                         │
        │         ████████████████████            │
        │         █    CEILING    █               │
        │         ████████████████████            │
        │         ████║         ║████             │
        │         ████║  WALL   ║████             │
        │         ████║ TEXTURE ║████             │
        │         ████║         ║████             │
        │         ████████████████████            │
        │         █▓▓▓▓▓▓▓▓▓▓▓▓▓▓█               │
        │         █▓▓ FLOOR ▓▓▓▓█                │
        │         █▓▓▓▓▓▓▓▓▓▓▓▓▓▓█               │
        └─────────────────────────────────────────┘

        Each █ or ▓ represents a pixel written by the rasterizer
        Different textures create different patterns

PERFORMANCE BREAKDOWN:
  Portal culling:      500 polygons → 50 visible (10× reduction)
  Edge building:       50 polygons × ~100 lines each = ~5,000 lines
  Texture precalc:     ~5,000 lines × 2 coordinates = ~10,000 calculations
  Pixel writing:       50 polygons × ~2,000 pixels each = ~100,000 pixels/frame
                       At 30 FPS = ~3,000,000 pixels/second

  Bottleneck: Pixel writing (inner loop)
  Solution: Assembly-optimized inner loops (68K/PowerPC)
```

### Shading System

**Shading Table Calculation**:
```c
#define CALCULATE_SHADING_TABLE(result, view, shading_tables, depth, ambient_shade)
{
    fixed shade;
    if (ambient_shade < 0) {
        // Self-luminescent (negative = absolute brightness)
        table_index = SHADE_TO_SHADING_TABLE_INDEX(-ambient_shade);
    } else {
        // Blend ambient and depth-based shading
        shade = view->maximum_depth_intensity - DEPTH_TO_SHADE(depth);
        shade = PIN(shade, 0, FIXED_ONE);
        table_index = SHADE_TO_SHADING_TABLE_INDEX(
            (ambient_shade > shade) ?
            (ambient_shade + (shade >> 1)) :  // Average
            (shade + (ambient_shade >> 1)));
    }
    
    result = shading_tables + table_index * MAXIMUM_SHADING_TABLE_INDEXES * sizeof(pixel);
}
```

**Shading Tables**:
- 8-bit: 32 tables × 256 colors
- 16-bit/32-bit: 64 tables × 256 entries
- Each entry maps palette index → lit pixel

### Sprite Rendering

**Object Sort** (`build_render_object_list()`):

Walks sorted polygons back-to-front, collecting objects:
```c
for (each sorted polygon) {
    for (each object in polygon) {
        build_render_object();
        sort_render_object_into_tree();  // Depth sort
    }
}
```

**Object Data**:
```c
struct render_object_data {
    struct sorted_node_data *node;
    struct clipping_window_data *clipping_windows;
    struct render_object_data *next_object;
    struct rectangle_definition rectangle;
    short ymedia;  // Media clipping boundary
};
```

**Rectangle Texture Mapping** (`texture_rectangle()` [scottish_textures.c:665]):

Projects 3D sprite bounds to screen rectangle, then renders column-by-column.

### Transfer Modes (Special Effects)

**Effect Types**:
```c
_textured_transfer        // Normal
_tinted_transfer         // Color overlay
_solid_transfer          // Solid color
_shadeless_transfer      // Ignores lighting
_static_transfer         // TV snow
_big_landscaped_transfer // Screen-space texture
```

**Examples**:
- `_xfer_invisibility` - Semi-transparent via shading
- `_xfer_fold_in/fold_out` - Teleport shrink
- `_xfer_fast_horizontal_slide` - Scrolling texture
- `_xfer_wobble` - Perspective distortion
- `_xfer_pulsate` - Breathing scale

### Edge Table Building (Bresenham's DDA Algorithm)

Marathon uses a modified Bresenham line algorithm to build edge tables - arrays of x or y coordinates for each scan line or column of a polygon.

**Why Edge Tables?**
Instead of computing polygon edges during rasterization, Marathon precomputes all edge coordinates into "scratch tables" before drawing. This separates the geometric work from the pixel-writing inner loop.

**Scratch Table Architecture**:
```c
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
┌─────────────────────────────────────────────────────────────────┐
│  dx = x1 - x0;  adx = |dx|;  dx = SGN(dx)                       │
│  dy = y1 - y0;  ady = |dy|;  dy = SGN(dy)                       │
│                                                                  │
│  if (adx >= ady)  // X-dominant line                            │
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
└─────────────────────────────────────────────────────────────────┘
```

**build_y_table()** - For vertical surfaces (walls):
```
Given edge from (x0,y0) to (x1,y1), builds table of y values for each x:

    Screen column x:
         200    201    202   ...   400
          │      │      │          │
          ▼      ▼      ▼          ▼
    y0: [100]  [102]  [104]  ... [150]  ← top_y per column
    y1: [300]  [298]  [296]  ... [250]  ← bottom_y per column

Same Bresenham algorithm, but recording y for each x step.
```

**Edge Table Usage in Polygon Rasterization**:
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
```

**Vertical Polygon (wall)**:
```
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

### Span Caching / Precalculation System

Marathon's key optimization is **precalculating texture mapping parameters** for entire scan lines or columns before the pixel-writing loop. This is stored in the `precalculation_table`.

**Precalculation Table**:
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

**Precalculation Flow**:
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

**Why This Matters**:
- Inner loop contains NO divides (perspective math done in precalc)
- Shading table lookup is single array index
- All branching decisions made before rasterization
- Cache-friendly memory access pattern

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

**Tree Building Process**:
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

**Node Tree vs Sorted Tree**:
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

Marathon's object sorting is surprisingly complex due to edge cases with overlapping objects in multiple polygons.

**Object Sorting Challenges** (from render.c comments):
1. Objects can overlap into polygons clipped behind them
2. Multiple non-overlapping objects with uncertain relative order
3. Objects below viewer projecting into higher polygons
4. Parasitic objects (players with attached items)

**Render Object Structure**:
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

**Object Sorting Algorithm**:
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

**Depth Ordering Within Node**:
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

### Polygon Clipping System

Marathon uses a hierarchical clipping system with endpoint clips, line clips, and clipping windows.

**Clipping Data Structures**:
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

**Clip Flags**:
```c
enum {
    _clip_left  = 0x0001,  // Clip against left edge
    _clip_right = 0x0002,  // Clip against right edge
    _clip_up    = 0x0004,  // Clip against top edge
    _clip_down  = 0x0008   // Clip against bottom edge
};
```

**Flagged World Points** (for clipping operations):
```c
struct flagged_world_point2d {  // For floors
    world_distance x, y;
    word flags;  // Which edges this point was clipped against
};

struct flagged_world_point3d {  // For ceilings/walls
    world_distance x, y, z;
    word flags;
};
```

**Clipping Algorithm**:
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
│    ┌──────────────┐       ┌──────────────┐                     │
│    │              │       │              │                     │
│    │    ┌────┐    │       │    Content   │                     │
│    │    │ P  │────┼───────┼──> clipped   │                     │
│    │    │ O  │    │       │    by portal │                     │
│    │    │ R  │    │       │    edges     │                     │
│    │    │ T  │    │       │              │                     │
│    │    │ A  │    │       │              │                     │
│    │    │ L  │    │       │              │                     │
│    │    └────┘    │       └──────────────┘                     │
│    └──────────────┘                                             │
│                                                                 │
│  Child inherits parent's clip window PLUS portal edges          │
│  Result: Nested clipping for arbitrary portal depth             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Clipping Window Building**:
```c
struct clipping_window_data *build_clipping_windows(
    struct view_data *view,
    struct node_data **node_list,
    short node_count
);

// Combines clipping from all paths to a polygon into unified windows
// Multiple paths may create multiple non-overlapping windows
// (e.g., visible through two separate portals)
```

### Key Constants

```c
MAXIMUM_NODES = 512                  // Portal tree depth
MAXIMUM_SORTED_NODES = 128           // Rendered polygons
MAXIMUM_RENDER_OBJECTS = 72          // Sprites
MAXIMUM_CLIPPING_WINDOWS = 256       // Portal clips
NORMAL_FIELD_OF_VIEW = 80           // Degrees
MINIMUM_OBJECT_DISTANCE = WORLD_ONE/20
```

---

## 6. Physics and Collision

> **🔧 For Porting:** `physics.c` and `world.c` are fully portable! No Mac dependencies. The collision detection uses pure fixed-point math against polygon geometry. Keep all code as-is.

### Fixed-Point Mathematics

**Core Constants**:
```c
#define FIXED_FRACTIONAL_BITS 16
#define FIXED_ONE (1<<16)              // 65536 = 1.0
#define WORLD_ONE 1024                 // 1 game unit
#define WORLD_FRACTIONAL_BITS 10
```

**Conversion**:
```c
INTEGER_TO_FIXED(x) = x << 16
FIXED_TO_INTEGER(x) = x >> 16
WORLD_TO_FIXED(x) = x << 6
FIXED_TO_WORLD(x) = x >> 6
```

### Player Physics Variables

```c
struct physics_variables {
    // Angular motion
    fixed head_direction;         // Free look
    fixed direction;              // Heading (yaw)
    fixed elevation;              // Pitch
    fixed angular_velocity;
    fixed vertical_angular_velocity;
    
    // Linear motion
    fixed velocity;               // Forward/back
    fixed perpendicular_velocity; // Strafe
    
    // Position (high precision)
    fixed_point3d position;       // 32-bit per axis
    fixed_point3d last_position;
    
    // Environment
    fixed floor_height, ceiling_height;
    fixed media_height;
    fixed actual_height;          // Player model height
    
    // External forces
    fixed_vector3d external_velocity;  // Knockback, current
    fixed external_angular_velocity;
    
    // Animation
    fixed step_phase;             // Walk cycle [0,1)
    fixed step_amplitude;
    
    word flags;                   // Above/below ground, in media
    short action;                 // Stationary, walking, running, airborne
};
```

### Physics Constants

**Walking Model**:
```c
maximum_forward_velocity = FIXED_ONE/14         // ~0.071 (2.1 units/sec)
maximum_backward_velocity = FIXED_ONE/17        // ~0.059
maximum_perpendicular_velocity = FIXED_ONE/20   // ~0.050

acceleration = FIXED_ONE/200                    // 0.005/tick
deceleration = FIXED_ONE/100                    // 0.010/tick
airborne_deceleration = FIXED_ONE/180           // ~0.0056/tick

gravitational_acceleration = FIXED_ONE/400      // 0.0025/tick
terminal_velocity = FIXED_ONE/7                 // ~0.143 (~4.3 units/sec)

angular_acceleration = 5*FIXED_ONE/8
maximum_angular_velocity = 6*FIXED_ONE
```

**Physics Constants Quick Reference**:

| Constant | Walking | Running | Units |
|----------|---------|---------|-------|
| Max Forward Velocity | FIXED_ONE/14 | FIXED_ONE/8 | per tick |
| Max Backward Velocity | FIXED_ONE/17 | FIXED_ONE/12 | per tick |
| Max Strafe Velocity | FIXED_ONE/20 | FIXED_ONE/13 | per tick |
| Acceleration | FIXED_ONE/200 | FIXED_ONE/100 | per tick² |
| Deceleration | FIXED_ONE/100 | FIXED_ONE/50 | per tick² |
| Airborne Deceleration | FIXED_ONE/180 | FIXED_ONE/180 | per tick² |
| Gravity | FIXED_ONE/400 | FIXED_ONE/400 | per tick² |
| Climbing Accel | FIXED_ONE/300 | FIXED_ONE/200 | per tick² |
| Terminal Velocity | FIXED_ONE/7 | FIXED_ONE/7 | per tick |

**Environment Modifiers**:

| Condition | Effect |
|-----------|--------|
| Low gravity | Gravity ÷ 2 |
| Feet in liquid | Gravity ÷ 2, Terminal velocity ÷ 2 |
| Head in liquid | Can swim upward |
| Airborne | Deceleration ÷ 4.5 (~FIXED_ONE/180) |

**Running Model**: ~2x velocity limits, ~2x acceleration, same gravity.

### Physics Update Loop

**Per Tick** (`physics_update()`):

1. **Horizontal Movement** (if grounded or in water):
   ```c
   if (moving_forward) {
       delta = (old_velocity < 0) ? 
               acceleration + deceleration :  // Reversing - snappy!
               acceleration;
       velocity = CEILING(velocity + delta, max_forward);
   } else {
       // Decelerate
       velocity = (velocity >= 0) ?
                  FLOOR(velocity - deceleration, 0) :
                  CEILING(velocity + deceleration, 0);
   }
   ```

2. **Vertical Motion**:
   ```c
   // Falling
   if (delta_z > 0) {
       gravity = gravitational_acceleration;
       if (environment_flags & _low_gravity) gravity >>= 1;
       if (feet_below_media) gravity >>= 1;
       external_velocity.k = FLOOR(external_velocity.k - gravity, -terminal_velocity);
   }
   
   // Climbing/jumping
   if (delta_z < 0) {
       external_velocity.k = CEILING(external_velocity.k + climbing_acceleration,
                                     terminal_velocity);
   }
   
   // Swimming
   if (head_below_media && swim_flag) {
       external_velocity.k += climbing_acceleration;
   }
   ```

3. **Position Update**:
   ```c
   cosine = cosine_table[direction >> FIXED_FRACTIONAL_BITS];
   sine = sine_table[direction >> FIXED_FRACTIONAL_BITS];
   
   new_position.x += (velocity * cosine - perpendicular_velocity * sine) >> TRIG_SHIFT;
   new_position.y += (velocity * sine + perpendicular_velocity * cosine) >> TRIG_SHIFT;
   new_position.z += external_velocity.k;
   
   // Add external forces (knockback, current)
   new_position.x += external_velocity.i;
   new_position.y += external_velocity.j;
   ```

4. **Ground Contact**:
   ```c
   if (landing_on_ground) {
       // Bounce absorption
       external_velocity.k /= (2 * COEFFICIENT_OF_ABSORBTION);  // COEF=2 → 1/4 velocity
   }
   
   if (hitting_ceiling) {
       external_velocity.k /= -COEFFICIENT_OF_ABSORBTION;  // Reverse + reduce
       new_z = ceiling_height - actual_height;
   }
   ```

5. **Friction on External Velocity**:
   ```c
   magnitude = isqrt(external_velocity.i² + external_velocity.j²);
   delta = (grounded) ? external_deceleration : external_deceleration >> 2;
   
   if (magnitude > ABS(delta)) {
       external_velocity.i -= (external_velocity.i * delta) / magnitude;
       external_velocity.j -= (external_velocity.j * delta) / magnitude;
   } else {
       external_velocity.i = external_velocity.j = 0;
   }
   ```

### Physics Models

Marathon supports multiple physics models that define how the player moves. The model is selected based on whether the player is walking or running.

**Physics Model Types**:
```c
enum {
    _model_game_walking,   // Normal movement
    _model_game_running,   // Shift held - faster movement
    NUMBER_OF_PHYSICS_MODELS
};
```

**Physics Constants Structure** [physics_models.h:17]:
```c
struct physics_constants {
    // Linear motion limits
    fixed maximum_forward_velocity;
    fixed maximum_backward_velocity;
    fixed maximum_perpendicular_velocity;

    // Linear motion rates
    fixed acceleration;
    fixed deceleration;
    fixed airborne_deceleration;
    fixed gravitational_acceleration;
    fixed climbing_acceleration;
    fixed terminal_velocity;
    fixed external_deceleration;

    // Angular motion
    fixed angular_acceleration;
    fixed angular_deceleration;
    fixed maximum_angular_velocity;
    fixed angular_recentering_velocity;
    fixed fast_angular_velocity;
    fixed fast_angular_maximum;
    fixed maximum_elevation;
    fixed external_angular_deceleration;

    // Step animation
    fixed step_delta;
    fixed step_amplitude;

    // Player dimensions
    fixed radius;
    fixed height;
    fixed dead_height;
    fixed camera_height;
    fixed splash_height;
    fixed half_camera_separation;
};
```

**Walking vs Running Constants**:
```
┌──────────────────────────────────────────────────────────────────────────┐
│                    PHYSICS MODEL COMPARISON                               │
├──────────────────────────────────┬───────────────────────────────────────┤
│           WALKING                │              RUNNING                   │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Max Forward:    FIXED_ONE/14     │ Max Forward:    FIXED_ONE/8           │
│                 (~4,681)         │                 (~8,192)  [1.75x]     │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Max Backward:   FIXED_ONE/17     │ Max Backward:   FIXED_ONE/12          │
│                 (~3,855)         │                 (~5,461)  [1.42x]     │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Max Strafe:     FIXED_ONE/20     │ Max Strafe:     FIXED_ONE/13          │
│                 (~3,276)         │                 (~5,041)  [1.54x]     │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Acceleration:   FIXED_ONE/200    │ Acceleration:   FIXED_ONE/100  [2x]   │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Deceleration:   FIXED_ONE/100    │ Deceleration:   FIXED_ONE/50   [2x]   │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Climb Accel:    FIXED_ONE/300    │ Climb Accel:    FIXED_ONE/200 [1.5x]  │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Angular Accel:  5*FIXED_ONE/8    │ Angular Accel:  5*FIXED_ONE/4  [2x]   │
├──────────────────────────────────┼───────────────────────────────────────┤
│ Max Angular:    6*FIXED_ONE      │ Max Angular:    10*FIXED_ONE  [1.67x] │
├──────────────────────────────────┴───────────────────────────────────────┤
│ Shared between both models (same values):                                │
├──────────────────────────────────────────────────────────────────────────┤
│ Gravity:           FIXED_ONE/400  (~164)                                 │
│ Terminal Velocity: FIXED_ONE/7    (~9,362)                               │
│ Airborne Decel:    FIXED_ONE/180  (~364)                                 │
│ Radius:            FIXED_ONE/4    (0.25 units)                           │
│ Height:            4*FIXED_ONE/5  (0.8 units)                            │
│ Camera Height:     1*FIXED_ONE/5  (0.2 units from top)                   │
│ Max Elevation:     QUARTER_CIRCLE*FIXED_ONE/3  (30 degrees)              │
└──────────────────────────────────────────────────────────────────────────┘
```

**Physics Model Selection**:
```c
static struct physics_constants *get_physics_constants_for_model(
    short physics_model,
    long action_flags
) {
    // Select walking or running based on action flags
    short model = (action_flags & _run_dont_walk) ?
                  _model_game_running :
                  _model_game_walking;

    // Apply map physics override if set
    if (physics_model != _editor_physics_model) {
        model = physics_model;
    }

    return &physics_models[model];
}
```

**Practical Effects**:
```
Walking:
  - Slower movement (14 units/sec forward)
  - Gentler acceleration (takes longer to reach max speed)
  - Lower turn rate
  - Better precision for combat

Running (Shift held):
  - ~1.75x faster forward movement
  - Snappier acceleration and deceleration
  - Higher turn rate
  - Harder to aim precisely
  - Same gravity and terminal velocity
```

**Network Synchronization Note**:
The physics model is determined by action flags, which are synchronized in multiplayer. This ensures all clients calculate identical physics regardless of local frame rate.

### Collision Detection

#### Player vs Walls (`keep_line_segment_out_of_walls()`)

**Algorithm**:
1. **Build exclusion zones** around lines and endpoints
2. **Multi-pass clipping**:
   - First pass: Clip against all colliding lines, record which ones
   - Second pass: Re-clip, accept only lines hit in first pass
   - Abort if new line detected (corner trap)
   - Point pass: Clip against endpoint circles

**Exclusion Zone**:
```c
struct side_exclusion_zone {
    world_point2d e0, e1;    // Line endpoints
    world_point2d e2, e3;    // Perpendicular expansion
};
```

Walls expanded by `MINIMUM_SEPARATION_FROM_WALL = WORLD_ONE/4`.

#### Exclusion Zone Visualization

**Line Expansion** (Creating Collision Boundary):

```
Original wall line:
         e0 ────────────────── e1
         (endpoint 0)    (endpoint 1)

Expanded exclusion zone (perpendicular expansion):

         e3 ────────────────── e2  ↑
          │                    │   │ MINIMUM_SEPARATION
          │   Exclusion Zone   │   │ (WORLD_ONE/4 = 256 units)
          │                    │   ↓
         e0 ────────────────── e1
          │                    │
          Player cannot        │
          enter this area      │

Quad vertices: [e0, e1, e2, e3] form collision boundary
```

**Point-in-Quad Test** (Cross-Product Method):

```
Test if player position P is inside exclusion zone:

    e3 ───────────────── e2
     │                   │
     │    ?  P           │
     │                   │
    e0 ───────────────── e1

For each edge (e0→e1, e1→e2, e2→e3, e3→e0):
    cross = (P.x - e0.x) × (e1.y - e0.y) - (P.y - e0.y) × (e1.x - e0.x)

All crosses must be positive (same orientation) for P to be inside

If inside: Project P onto nearest edge to push out
```

**Multi-Pass Clipping Algorithm**:

```
Initial player movement:

    Start                           Goal
      @──────────────────────────────>*
                  │
                Wall

Pass 1: Find all lines that could collide
    Lines detected: [Line 5, Line 7, Line 12]

Pass 2: Re-clip using only lines from Pass 1
    Verify collisions are consistent
    If new line appears → Corner trap! Abort movement

Pass 3: Clip against endpoint circles
    Endpoints act as rounded corners
    Distance test: dist(player, endpoint) < (player_radius + wall_separation)

Final position:
      @───────────>*
                  │  (Clipped to stay outside wall)
                Wall
```

**Height Validation** (3D Collision):

```
Side view of wall crossing:

  Ceiling ──────────────────────
                    │ Adjacent
    Player →        │ polygon
    height          │ ceiling
                    │
  ──────────────────┼───────────  ← Step height check
  Floor             │
                    │ Adjacent
                    │ polygon
                    │ floor
  ──────────────────────────────

Collision checks:
1. adjacent_floor - player.z > maximum_delta_height?
   → Too high to step up (can't climb)

2. adjacent_ceiling - player.z < player_height?
   → Head would hit ceiling (blocked)

3. (adjacent_ceiling - adjacent_floor) < player_height?
   → Space too narrow (can't fit)

Maximum step height ≈ WORLD_ONE/3 (~341 units)
```

#### Player vs Objects (`legal_player_move()`)

**Bounding Cylinder Test** (2D + 3D):

```c
for (each nearby object) {
    // 2D cylinder test
    separation = player_radius + obstacle_radius;
    new_distance² = (new_dx)² + (new_dy)²;

    if (new_distance² < separation² && approaching) {
        // 3D overlap check
        if (new_z + player_height >= obstacle_z &&
            new_z <= obstacle_z + obstacle_height) {
            return obstacle_index;  // Collision!
        } else if (obstacle_z + obstacle_height > floor) {
            floor = obstacle_z + obstacle_height;  // Platform
        }
    }
}
```

#### Bounding Box Collision Visualization

**Top-Down View** (2D Cylinder Test):

```
Player attempting to move toward object:

    Start position              Goal position
         @                             *
     ┌───┴───┐                    ┌────┴────┐
     │Player │                    │ Player  │
     │Radius │                    │ Radius  │
     └───┬───┘                    └────┬────┘
         │                             │
         └─────────────────────────────┘
                Movement vector

                    ┌──────┐
                    │Object│  ← Obstacle in path
                    │Radius│
                    └──────┘

Collision test:
1. Calculate distance from new position to object center
   distance² = (new_x - obj_x)² + (new_y - obj_y)²

2. Required separation = player_radius + object_radius

3. If distance² < separation²:
   → Cylinders overlap (potential collision)
```

**Side View** (3D Height Test):

```
Scenario A: Collision (vertical overlap)

  Player         Object
    ┌──┐         ┌────┐
    │  │ height  │    │ height
    │@─┼─────────┼──X │  ← Heights overlap!
    │  │         │    │
    └──┴─────────┴────┘
    floor        floor

    Test: player.z + player.height >= object.z
          AND player.z <= object.z + object.height
    Result: COLLISION

Scenario B: No collision (player above object)

    ┌──┐
    │@ │  Player standing on platform
    └──┴─────────┬────┐
                 │    │  Object below
                 └────┘

    Test: player.z > object.z + object.height
    Result: NO COLLISION (player is above)

Scenario C: Object as platform

                 ┌──┐
                 │@ │  ← Player on top
    ─────────────┴──┴───────
                 ┌────┐
                 │Obj │  Becomes floor
                 └────┘

    If object.z + object.height > current_floor:
        floor = object.z + object.height
    Player can stand on objects!
```

**Circle-Circle Distance**:

```
Fast distance check (avoids square root):

    Player               Object
      @ ───────d──────── ⊕

Required: d < (r1 + r2)
Squared:  d² < (r1 + r2)²

Code optimization:
    distance_squared = dx² + dy²
    separation_squared = (r1 + r2)²

    if (distance_squared < separation_squared) {
        // Collision!
    }

Avoids expensive sqrt() call
```

#### Projectile Physics (`translate_projectile()`)

**Movement Per Tick**:
```c
// Apply gravity
if (affected_by_gravity) {
    projectile->gravity -= GRAVITATIONAL_ACCELERATION;
}

// Update Z
new_location.z += projectile->gravity;

// Translate along facing/elevation
translate_point3d(&new_location, speed, facing, elevation);

// Wander (accuracy spray)
if (horizontal_wander) {
    translate_point3d(&new_location,
                     (random() & 1) ? WANDER_MAGNITUDE : -WANDER_MAGNITUDE,
                     NORMALIZE_ANGLE(facing + QUARTER_CIRCLE), 0);
}
```

#### Projectile Movement Visualization

**Bullet Trajectory** (Gravity + Velocity):

```
Frame-by-frame projectile movement (e.g., grenade):

Frame 0: Launch
         @ ──────>  Initial velocity
         │
         └ Facing direction = 45° up
           Speed = 20 units/tick

Frame 1:
            *  Position after 1 tick
           ╱   velocity.z still positive
          ╱    gravity starts pulling down
         @

Frame 5:
               *  Apex (velocity.z ≈ 0)
              ╱
             ╱
            ╱
           ╱
          @

Frame 10:
                    *  Falling (velocity.z negative)
                   ╱
                  ╱   gravity accumulated
                 ╱
                ╱
         ──────────────  ← Floor
         Impact!

Physics each tick:
    velocity.z -= GRAVITATIONAL_ACCELERATION
    position.z += velocity.z
    position.x += speed * cos(facing)
    position.y += speed * sin(facing)
```

**Accuracy Spray** (Wander):

```
Perfect aim (no wander):
    Weapon → ─────────────> All bullets hit same point

With horizontal wander:
    Weapon → ─────╱───────> Bullet deviates left/right
             ─────────────>
             ─────╲───────>

Wander implementation:
    Each tick: random perpendicular offset
    Magnitude: WANDER_MAGNITUDE (small value)
    Direction: facing + 90° (perpendicular)

Result: Cone of fire for automatic weapons
```

**Collision Loop**:
```c
do {
    // Check polygon boundary crossing
    line_index = find_line_crossed_leaving_polygon(current_polygon,
                                                   old_location, new_location);
    if (line_index != NONE) {
        // Calculate intersection
        find_line_intersection(&e0, &e1, old_location, new_location, &intersection);

        // Solid wall?
        if (LINE_IS_SOLID(line)) {
            contact = _hit_wall;
        } else {
            // Check adjacent polygon vertical clearance
            if (intersection.z > adjacent_floor &&
                intersection.z < adjacent_ceiling) {
                current_polygon = adjacent_polygon;
            } else {
                contact = _hit_wall;
            }
        }
    }

    // Check floor/ceiling in current polygon
    if (new_z < polygon->floor_height) contact = _hit_floor;
    if (new_z > polygon->ceiling_height) contact = _hit_ceiling;

    // Check objects (monsters, scenery)
    possible_intersecting_monsters(...);

} while (line_index != NONE && contact == _hit_nothing);
```

#### Projectile Polygon Traversal

**Line Crossing Detection**:

```
Projectile moving across polygons:

    Polygon A         │     Polygon B
                     Line
    Old pos          │         New pos
       @─────────────┼───────────>*
                     │
                Intersection point

Algorithm:
1. Test projectile path against all edges of current polygon
2. If crosses edge:
   - Calculate exact intersection point
   - Check if line is solid → hit wall
   - Check vertical clearance in adjacent polygon
   - If passable: move to adjacent polygon, continue

Path through multiple polygons:

    @─────→│────→│────→│──→* Hit!
    Poly 0 │Poly1│Poly2│Wall
           ↓     ↓     ↓
    Find   Cont. Cont. Stop
    cross  check check (solid)
```

**Vertical Clearance Check**:

```
Side view of projectile crossing portal:

  Ceiling A ─────────┐     ┌───── Ceiling B
                     │     │
    Projectile       Portal
       ──>           │     │
                     │     │
  Floor A ──────────┐└─────┴───── Floor B
                     Too low!

Test at intersection point:
    if (z > adjacent_floor && z < adjacent_ceiling):
        → Can pass through portal
    else:
        → Hit floor or ceiling of portal
```

**Bounce Physics** (Grenades):

```
Grenade with _rebounds_from_floor flag:

Impact with floor:
       ╲
        ╲  velocity.z = -500 (falling)
         *────  Floor

Bounce:
       ╱   velocity.z = +400 (reversed, reduced)
      ╱    Energy absorbed by bounce
     *────  Floor

Code:
    if (contact == _hit_floor && rebounds_from_floor) {
        velocity.z = -velocity.z * BOUNCE_COEFFICIENT;
        // Continue moving
    }
```

**Projectile Flags**:
- `_guided` - Homing
- `_affected_by_gravity` - Standard gravity
- `_affected_by_half_gravity` - Reduced (rockets)
- `_doubly_affected_by_gravity` - Heavy (grenades)
- `_rebounds_from_floor` - Bounce
- `_persistent` - Doesn't vanish on hit
- `_penetrates_media` - Ignores liquid
- `_horizontal_wander` / `_vertical_wander` - Accuracy spray

### Media Effects

**Water/Lava Physics**:
```c
if (feet_below_media) {
    gravity >>= 1;              // Half gravity
    terminal_velocity >>= 1;    // Half terminal velocity
}

// Current
if (feet_below_media) {
    apply_current_velocity_in_direction(media->current_direction);
}

// Swimming
if (head_below_media && swim) {
    external_velocity.k += climbing_acceleration;
}
```

---

## 7. Game Loop and Timing

> **🔧 For Porting:** `marathon2.c` contains the main game loop and is mostly portable. Replace the Mac event handling in `shell.c` with your window system's event loop. The core `update_world()` function needs no changes—just call it at 30 Hz with proper action flags.

> **For complete frame lifecycle**: See Section 32 (Life of a Frame) for detailed diagrams showing input → update → render flow, timing budgets, and data flow examples.

### The 30 Hz Main Loop

**Fundamental Constant**:
```c
#define TICKS_PER_SECOND 30
```

**Main Loop** (`update_world()`):

```
1. Input Processing - Gather action flags
2. For each queued tick:
   a. update_lights()           - Animate lighting
   b. update_medias()           - Liquid level, damage
   c. update_platforms()        - Elevators, doors
   d. update_control_panels()   - Terminals, switches
   e. update_players()          - Player physics
   f. move_projectiles()        - Projectile paths
   g. move_monsters()           - AI, attacks
   h. update_effects()          - Particles, explosions
   i. recreate_objects()        - Respawn items
   j. handle_random_sound_image() - Ambient sounds
   k. animate_scenery()         - Static objects
   l. update_net_game()         - Network sync
   m. Check level completion
3. Render Update - HUD, interface
4. Record/Replay - Store action flags
```

**Critical Design**: All physics updates happen within the same tick, then rendering occurs. This ensures:
- Network consistency (same inputs → same state)
- Deterministic replay
- No visual glitches

### Action Flags (Input Encoding)

**32-bit Encoding**:
```c
#define GET_ABSOLUTE_YAW(flags)       ((flags >> 7) & 0x7F)
#define GET_ABSOLUTE_PITCH(flags)     ((flags >> 14) & 0x1F)
#define GET_ABSOLUTE_POSITION(flags)  ((flags >> 22) & 0x7F)

// Flags:
_turning_left, _turning_right
_looking_left, _looking_right
_looking_up, _looking_down, _looking_center
_moving_forward, _moving_backward
_sidestepping_left, _sidestepping_right
_run_dont_walk
_left_trigger_state, _right_trigger_state
_action_trigger_state
_cycle_weapons_forward, _cycle_weapons_backward
_toggle_map, _swim
```

Compresses full 6-DOF input into 32 bits for network transmission.

---

## 8. Entity Systems

> **🔧 For Porting:** All entity code is fully portable! `monsters.c`, `projectiles.c`, `weapons.c`, `items.c`, `effects.c`, and `scenery.c` have no Mac dependencies. The definition headers (`*_definitions.h`) contain static data tables that compile anywhere.

### Monster System

**47 Monster Types** defined in `monster_definitions.h`:

**Factions**:
- **Pfhor** - Fighters, Troopers, Hunters, Enforcers, Juggernauts
- **Compilers** - AI entities, teleport, invisible variants
- **Cyborgs** - Projectile, flamethrower
- **Humans** - Crew, scientists, security (ally)
- **Native** - Ticks, Yetis (water/sewage/lava variants)

**Monster Definition** [monster_definitions.h:147] (128 bytes):
```c
struct monster_definition {
    short collection;                  // Sprite set
    short vitality;                    // Hit points
    unsigned long immunities;          // Damage types ignored
    unsigned long weaknesses;          // Amplified damage
    unsigned long flags;
    
    long monster_class, friends, enemies;  // Faction relationships
    
    world_distance radius, height;     // Collision bounds
    world_distance visual_range, dark_visual_range;
    short half_visual_arc, half_vertical_visual_arc;
    short intelligence;                // Pathfinding quality
    short speed, gravity, terminal_velocity;
    
    short attack_frequency;
    struct attack_definition melee_attack, ranged_attack;
};
```

**Monster AI** (`move_monsters()`):

1. **Activation**:
   - Player within activation range (5-8 WORLD_ONE)
   - Sound + line of sight
   - Glue trigger - activates zone recursively
   - Difficulty modifies spawn rate

2. **Target Acquisition**:
   - Uses class relationship matrix (friends/enemies)
   - Line of sight check
   - Distance evaluation
   - Dark environments use reduced range

3. **Pathfinding**:
   - Flood fill from monster to target
   - Respects movement constraints
   - Cost function for obstacles
   - Regenerates if blocked or target moves >2 WORLD_ONE
   - **Only one monster gets expensive AI per frame** (load distribution)

4. **Combat**:
   - Attack frequency: 2× per second (stationary), slower when moving
   - Melee range: ~1 WORLD_ONE
   - Projectile range: 10-25 WORLD_ONE
   - Error angle: Random spread (0-5 degrees typical)
   - Attack executes at animation keyframe

5. **Death**:
   - Soft death: Collapse animation
   - Hard death: Explosion/gibs
   - Flaming death: Environmental
   - Shrapnel damage on death (some monsters)
   - Random item drop

**Monster AI State Machine**:

```
Monster Action States (from monsters.h:129-142):

                                    ┌──────────────────────────────┐
                                    │                              │
                                    ▼                              │
                            ┌───────────────┐                      │
      ┌────────────────────►│  _stationary  │◄─────────────────────┤
      │                     └───────┬───────┘                      │
      │                             │                              │
      │                    See target / Path found                 │
      │                             │                              │
      │                             ▼                              │
      │                     ┌───────────────┐         No path      │
      │    ┌───────────────►│   _moving     │─────────────────────►│
      │    │                └───────┬───────┘                      │
      │    │                        │                              │
      │    │              In attack range?                         │
      │    │                        │                              │
      │    │            ┌───────────┴───────────┐                  │
      │    │            │                       │                  │
      │    │    Melee range              Ranged distance           │
      │    │            │                       │                  │
      │    │            ▼                       ▼                  │
      │    │   ┌─────────────────┐     ┌─────────────────┐         │
      │    │   │ _attacking_close│     │ _attacking_far  │         │
      │    │   └────────┬────────┘     └────────┬────────┘         │
      │    │            │                       │                  │
      │    │            └───────────┬───────────┘                  │
      │    │                        │                              │
      │    │                Attack complete                        │
      │    │                        │                              │
      │    │                        ▼                              │
      │    │           ┌───────────────────────────┐               │
      │    └───────────│ _waiting_to_attack_again  │───────────────┘
      │                └───────────────────────────┘
      │                       Cooldown done → back to _moving
      │
      │
Hit by damage                                    Killed
      │                                             │
      ▼                                             ▼
┌───────────────┐                    ┌──────────────────────────────────┐
│  _being_hit   │                    │         Death States             │
└───────┬───────┘                    │  ┌────────────────────────────┐  │
        │                            │  │ _dying_soft (collapse)     │  │
  Stun recovery                      │  ├────────────────────────────┤  │
        │                            │  │ _dying_hard (explosion)    │  │
        └────────────────────────────│  ├────────────────────────────┤  │
              (back to _moving)      │  │ _dying_flaming (fire death)│  │
                                     │  └────────────────────────────┘  │
                                     └──────────────────────────────────┘
                                                     │
                                                     ▼
                                              Monster removed


Teleport-capable monsters (Compilers):

┌──────────────────┐     ┌───────────────┐     ┌───────────────────┐
│ _teleporting_in  │────►│  _teleporting │────►│ _teleporting_out  │
└──────────────────┘     └───────────────┘     └───────────────────┘
    (fade in)            (active/invisible)        (fade out)


State Transition Triggers:
  _stationary → _moving:       Path found to target
  _moving → _attacking_*:      Target in attack range
  _attacking_* → _waiting:     Attack animation complete
  _waiting → _moving:          Attack cooldown expired
  Any state → _being_hit:      Took non-lethal damage
  _being_hit → _moving:        Stun recovery time elapsed
  Any state → _dying_*:        Took lethal damage (type determines death style)
```

#### Monster Mode System (Target Lock)

Separate from action states, monsters have a **mode** that tracks target lock status:

```c
enum /* monster modes */
{
    _monster_locked,       // Has valid target in sight
    _monster_losing_lock,  // Target moved, searching
    _monster_lost_lock,    // Target out of sight
    _monster_unlocked,     // No target
    _monster_running       // Fleeing (civilians)
};
```

**Mode Transition Diagram**:

```
                              Target acquired
                                    │
                                    ▼
                        ┌───────────────────┐
          ┌────────────►│  _monster_locked  │◄─────────────┐
          │             └─────────┬─────────┘              │
          │                       │                        │
          │               Target moved to                  │
          │               another polygon                  │
          │                       │                        │
    Target visible               ▼                    Target visible
    again              ┌─────────────────────┐        again
          │            │ _monster_losing_lock│             │
          │            └─────────┬───────────┘             │
          │                      │                         │
          │              Changed polygons                  │
          │              > intelligence times              │
          │                      │                         │
          │                      ▼                         │
          │            ┌─────────────────────┐             │
          └────────────│  _monster_lost_lock │─────────────┘
                       └─────────┬───────────┘
                                 │
                         No target found
                         after search
                                 │
                                 ▼
                       ┌─────────────────────┐
                       │  _monster_unlocked  │
                       └─────────────────────┘

Intelligence determines lock persistence:
  _intelligence_low = 2     polygon changes before losing lock
  _intelligence_average = 3 polygon changes before losing lock
  _intelligence_high = 8    polygon changes before losing lock
```

#### Monster Classes and Factions

**Class Bitmask System** (used for friends/enemies relationships):

| Class | Bit | Members |
|-------|-----|---------|
| `_class_player` | 0 | Player marine |
| `_class_human_civilian` | 1 | Crew, scientists, security |
| `_class_madd` | 2 | Rampant BOBs |
| `_class_possessed_hummer` | 3 | Durandal's hummers |
| `_class_defender` | 4 | Defenders |
| `_class_fighter` | 5 | Pfhor fighters |
| `_class_trooper` | 6 | Pfhor troopers |
| `_class_hunter` | 7 | Hunters |
| `_class_enforcer` | 8 | Enforcers |
| `_class_juggernaut` | 9 | Juggernauts |
| `_class_hummer` | 10 | Hummers |
| `_class_compiler` | 11 | Compilers |
| `_class_cyborg` | 12 | Cyborgs |
| `_class_assimilated_civilian` | 13 | Assimilated BOBs |
| `_class_tick` | 14 | Ticks |
| `_class_yeti` | 15 | Yetis |

**Faction Groupings**:
```c
_class_human    = _class_player | _class_human_civilian | _class_madd | _class_possessed_hummer
_class_pfhor    = _class_fighter | _class_trooper | _class_hunter | _class_enforcer | _class_juggernaut
_class_client   = _class_compiler | _class_assimilated_civilian | _class_cyborg | _class_hummer
_class_native   = _class_tick | _class_yeti
```

**Attitude Calculation**:
```c
short get_monster_attitude(short monster_index, short target_index) {
    if (definition->friends & target_class) return _friendly;
    if (definition->enemies & target_class) return _hostile;
    return _neutral;
}
```

#### Monster Flags Reference

| Flag | Value | Description |
|------|-------|-------------|
| `_monster_is_omniscent` | 0x1 | Ignores line-of-sight for targeting |
| `_monster_flys` | 0x2 | Can move vertically freely |
| `_monster_is_alien` | 0x4 | Slower on easier difficulties |
| `_monster_major` | 0x8 | Type-1 is minor variant |
| `_monster_minor` | 0x10 | Type+1 is major variant |
| `_monster_cannot_be_dropped` | 0x20 | Always spawns regardless of difficulty |
| `_monster_floats` | 0x40 | Gradual vertical movement |
| `_monster_cannot_attack` | 0x80 | Runs to safety (civilians) |
| `_monster_uses_sniper_ledges` | 0x100 | Positions on elevated platforms |
| `_monster_is_invisible` | 0x200 | Uses invisibility transfer mode |
| `_monster_is_subtly_invisible` | 0x400 | Partial invisibility |
| `_monster_is_kamakazi` | 0x800 | Suicides when close to target |
| `_monster_is_berserker` | 0x1000 | Goes berserk below 1/4 vitality |
| `_monster_is_enlarged` | 0x2000 | 1.25× normal height |
| `_monster_has_delayed_hard_death` | 0x4000 | Soft death then hard death |
| `_monster_fires_symmetrically` | 0x8000 | Fires at ±dy simultaneously |
| `_monster_has_nuclear_hard_death` | 0x10000 | Screen flash on death |
| `_monster_cant_fire_backwards` | 0x20000 | Max 135° turn to fire |
| `_monster_can_die_in_flames` | 0x40000 | Uses flaming death animation |
| `_monster_waits_with_clear_shot` | 0x80000 | Holds position if has clear shot |
| `_monster_is_tiny` | 0x100000 | 0.25× normal height |
| `_monster_attacks_immediately` | 0x200000 | No delay before first attack |

#### Monster Speed Table

| Speed Constant | Value | Units/Second |
|----------------|-------|--------------|
| `_speed_slow` | WORLD_ONE/120 | ~8.5 |
| `_speed_medium` | WORLD_ONE/80 | ~12.8 |
| `_speed_almost_fast` | WORLD_ONE/70 | ~14.6 |
| `_speed_fast` | WORLD_ONE/40 | ~25.6 |
| `_speed_superfast1` | WORLD_ONE/30 | ~34.1 |
| `_speed_blinding` | WORLD_ONE/20 | ~51.2 |
| `_speed_insane` | WORLD_ONE/10 | ~102.4 |

#### AI Load Distribution System

Marathon distributes expensive AI operations across frames to maintain performance:

```
Frame N:    Monster 0 gets target search time
Frame N+1:  Monster 1 gets target search time
Frame N+2:  Monster 2 gets target search time
...

Frame N:    Monster 0 gets pathfinding time
Frame N+4:  Monster 1 gets pathfinding time  (pathfinding is 1 per 4 frames)
Frame N+8:  Monster 2 gets pathfinding time
...
```

**Key Variables**:
```c
dynamic_world->last_monster_index_to_get_time   // Round-robin for targeting
dynamic_world->last_monster_index_to_build_path // Round-robin for pathfinding
```

**Rules**:
- Only ONE monster gets expensive target search per frame
- Only ONE monster gets pathfinding per 4 frames
- Monsters without paths get immediate pathfinding regardless
- When all monsters have had a turn, index resets to -1

#### Activation System

**Activation Ranges**:
```c
GLUE_TRIGGER_ACTIVATION_RANGE = 8 * WORLD_ONE   // Trigger activation
MONSTER_ALERT_ACTIVATION_RANGE = 5 * WORLD_ONE  // Sound/sight activation
```

**Activation Biases** (set in editor):
```c
_activate_on_player           // Target player immediately
_activate_on_nearest_hostile  // Find closest enemy
_activate_on_goal             // Move toward goal polygon
_activate_randomly            // Random behavior
```

**Activation Flags**:
```c
_pass_one_zone_border            // Can cross one zone
_passed_zone_border              // Has crossed a zone
_activate_invisible_monsters     // Sound/teleport trigger
_activate_deaf_monsters          // Trigger (not sound)
_pass_solid_lines                // Trigger (not sound)
_use_activation_biases           // Follow editor instructions
_activation_cannot_be_avoided    // Cannot be suppressed
```

**Activation Flood Algorithm**:

```
1. Start flood from caller's polygon
2. For each polygon reached:
   - Check all objects in polygon
   - If object is monster and meets criteria:
     - Activate if inactive
     - Lock on target if hostile
     - Propagate activation to neighbors
3. Continue until maximum cost or no more polygons

Flood respects:
  - Zone boundaries (unless explicitly crossing)
  - Line-of-sight for sound activation
  - Deaf/blind monster flags
```

#### Pathfinding System

Marathon's pathfinding uses a two-layer system: a flood fill algorithm for exploring polygon connectivity, and a path builder that extracts waypoints for navigation.

**System Limits**:
```c
#define MAXIMUM_PATHS 20              // Concurrent paths (pathfinding.c)
#define MAXIMUM_POINTS_PER_PATH 63    // Waypoints per path
#define MAXIMUM_FLOOD_NODES 255       // Search nodes (flood_map.c)
```

##### Data Structures

**Flood Fill Node** [flood_map.c:39]:
```c
struct node_data {           // 16 bytes
    word flags;              // NODE_IS_EXPANDED bit flag
    short parent_node_index; // For backtracking
    short polygon_index;     // This polygon
    long cost;               // Accumulated cost to reach here
    short depth;             // Polygons from start
    long user_flags;         // Caller-defined flags
};
```

**Path Definition** (`pathfinding.c`):
```c
struct path_definition {     // 256 bytes
    short current_step;      // Current waypoint index
    short step_count;        // Total waypoints (NONE = free)
    world_point2d points[MAXIMUM_POINTS_PER_PATH];  // Waypoint coordinates
};
```

##### Flood Fill Modes

The flood map supports four search strategies:

| Mode | Description | Use Case |
|------|-------------|----------|
| `_best_first` | Expands lowest-cost node first | Optimal path, slower |
| `_breadth_first` | Expands in order added | Faster for large areas |
| `_flagged_breadth_first` | Breadth-first with user flags | Special constraints |
| `_depth_first` | Deepest node first | Not implemented |

```
_best_first (A* style):          _breadth_first (BFS):

  Expand lowest cost →             Expand in order added →
  ┌───┐                            ┌───┐
  │ 3 │ ← Skip                     │ 1 │ ← Expand first
  ├───┤                            ├───┤
  │ 1 │ ← Expand!                  │ 2 │ ← Expand second
  ├───┤                            ├───┤
  │ 5 │ ← Skip                     │ 3 │ ← Expand third
  └───┘                            └───┘
```

##### Cost Function

```c
typedef long (*cost_proc_ptr)(
    short source_polygon_index,
    short line_index,
    short destination_polygon_index,
    void *caller_data
);

// Monster pathfinding costs:
MONSTER_PATHFINDING_OBSTRUCTION_COST = 2 * WORLD_ONE²  // Objects blocking
MONSTER_PATHFINDING_PLATFORM_COST = 4 * WORLD_ONE²     // Moving platforms
MINIMUM_MONSTER_PATHFINDING_POLYGON_AREA = WORLD_ONE   // Polygon too small
```

**Cost function returns**:
- Positive value: Add to path cost
- Zero or negative: Polygon not traversable (blocked)
- NULL cost_proc: Use polygon area as cost (fastest)

##### Path Creation Flow

```
new_path(source, destination, cost_func)
                │
                ▼
    ┌──────────────────────────────────────┐
    │ flood_map(source_polygon, ...)       │
    │ Start flood fill from source         │
    └───────────────────┬──────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────┐
    │ while (polygon != destination)       │
    │   flood_map(NONE, ...)               │
    │   Expand next lowest-cost node       │
    └───────────────────┬──────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────┐
    │ reverse_flood_map()                  │
    │ Backtrack via parent_node_index      │
    │ Build path from destination to source│
    └───────────────────┬──────────────────┘
                        │
                        ▼
    ┌──────────────────────────────────────┐
    │ calculate_midpoint_of_shared_line()  │
    │ For each polygon transition:         │
    │ Generate waypoint on shared edge     │
    └──────────────────────────────────────┘
```

##### Waypoint Generation

Waypoints are placed on the shared edge between adjacent polygons:

```
  ┌─────────────────┐
  │  Polygon A      │
  │       ●───────────────────► Monster path
  │       │         │
  └───────┼─────────┘
          │ ← Waypoint on shared line
  ┌───────┼─────────┐
  │       │         │
  │       ●         │
  │  Polygon B      │
  └─────────────────┘
```

The `calculate_midpoint_of_shared_line()` function finds the midpoint of the shared edge, respecting a minimum separation from walls.

##### Random Paths

When destination is NONE, a random path is generated:
1. Flood fill expands as far as possible
2. `choose_random_flood_node(bias)` picks random expanded node
3. If bias vector provided, prefer nodes in that direction
4. Useful for fleeing/wandering behavior

##### Path Following

```c
// Get next waypoint
boolean move_along_path(short path_index, world_point2d *p) {
    if (current_step < step_count) {
        *p = points[current_step++];
        return TRUE;
    }
    return FALSE;  // Path complete
}
```

**Path Invalidation Triggers**:
- Target moves more than 2 WORLD_ONE
- Monster completes attack
- Monster takes damage (stun recovery)
- Path blocked by closed door

#### Combat System Details

**Attack Frequency**:
- Stationary monsters: 2× attacks/second typical
- Moving monsters: Half attack rate
- Varies by monster type (1-4 seconds between attacks)

**Attack Definition Structure**:
```c
struct attack_definition {
    short type;                    // Projectile type
    short repetitions;             // Shots per attack
    angle error;                   // ± accuracy spread
    world_distance range;          // Maximum attack range
    short attack_shape;            // Animation keyframe for attack
    world_distance dx, dy, dz;     // Projectile spawn offset
};
```

**Attack Timing**:
```
Animation starts
    │
    ▼ (frames pass)
    │
Keyframe reached ──────► Projectile spawned / Damage dealt
    │
    ▼ (more frames)
    │
Animation ends ──────► Cooldown begins
    │
    ▼ (cooldown ticks)
    │
Ready to attack again
```

**Melee vs Ranged Selection**:
- If within melee range and has melee attack → melee
- If within ranged range and has ranged attack → ranged
- `_monster_chooses_weapons_randomly` flag randomizes selection

### Weapon System

**Weapon Classes**:
```c
_melee_class              // Fist
_normal_class             // Single trigger, one ammo type
_dual_function_class      // Primary/secondary different
_twofisted_pistol_class   // Dual wield
_multipurpose_class       // Rifle + grenade launcher
```

**Weapon Types** (`weapons.h:9-25`):

| ID | Enum Constant | Weapon | Ammo/Mag | Notes |
|----|---------------|--------|----------|-------|
| 0 | `_weapon_fist` | Fist | ∞ | Melee only |
| 1 | `_weapon_pistol` | Magnum Pistol | 8 | Can dual-wield |
| 2 | `_weapon_plasma_pistol` | Fusion Pistol | 20 | Chargeable |
| 3 | `_weapon_assault_rifle` | MA-75B Rifle | 52 | +7 grenades secondary |
| 4 | `_weapon_missile_launcher` | SPNKR-X17 | 2 | Guided missiles |
| 5 | `_weapon_flamethrower` | TOZT-7 | 210 ticks | Continuous flame |
| 6 | `_weapon_alien_shotgun` | Alien Weapon | Varies | Dropped by enemies |
| 7 | `_weapon_shotgun` | WSTE-M5 | 2 | Can dual-wield |
| 8 | `_weapon_ball` | Ball | - | Multiplayer only |
| 9 | `_weapon_smg` | KKV-7 SMG | 32 | Burst fire |

```c
// From weapons.h
enum { /* Weapons */
    _weapon_fist,                    // 0
    _weapon_pistol,                  // 1
    _weapon_plasma_pistol,           // 2
    _weapon_assault_rifle,           // 3
    _weapon_missile_launcher,        // 4
    _weapon_flamethrower,            // 5
    _weapon_alien_shotgun,           // 6
    _weapon_shotgun,                 // 7
    _weapon_ball,                    // 8
    _weapon_smg,                     // 9
    MAXIMUM_NUMBER_OF_WEAPONS,       // 10

    _weapon_doublefisted_pistols = MAXIMUM_NUMBER_OF_WEAPONS,  // 10 (pseudo)
    _weapon_doublefisted_shotguns,                              // 11 (pseudo)
    PLAYER_TORSO_SHAPE_COUNT                                    // 12
};
```

**Weapon Definition** [weapon_definitions.h:169] (196 bytes):
```c
struct weapon_definition {
    short weapon_class, flags;
    
    fixed idle_height, bob_amplitude, kick_height, reload_height;
    
    short collection;
    short idle_shape, firing_shape, reloading_shape;
    
    short ready_ticks, await_reload_ticks, loading_ticks;
    
    struct trigger_definition weapons_by_trigger[2];  // Primary/secondary
};

struct trigger_definition {
    short rounds_per_magazine;
    short ammunition_type;
    short ticks_per_round;         // Fire rate
    short recovery_ticks;
    short projectile_type;
    short theta_error;             // Spread angle
    short burst_count;             // Pellets per shot
};
```

**Weapon State Machine**:
```c
_weapon_idle, _weapon_raising, _weapon_lowering
_weapon_charging, _weapon_charged
_weapon_firing, _weapon_recovering
_weapon_awaiting_reload, _weapon_waiting_to_load
_weapon_finishing_reload
```

**Weapon State Machine Diagram**:

```
                              ┌──────────────────────────────────────────────────────┐
                              │                                                      │
                              ▼                                                      │
                      ┌───────────────┐                                              │
                      │  _weapon_idle │◄──────────────────────────────────┐          │
                      └───────┬───────┘                                   │          │
                              │                                           │          │
          ┌───────────────────┼───────────────────┐                       │          │
          │                   │                   │                       │          │
    Switch weapon      Fire pressed        Need reload                    │          │
          │                   │                   │                       │          │
          ▼                   ▼                   ▼                       │          │
  ┌───────────────┐   ┌───────────────┐   ┌─────────────────┐             │          │
  │   _lowering   │   │   _charging   │   │ _awaiting_reload│             │          │
  └───────┬───────┘   └───────┬───────┘   └────────┬────────┘             │          │
          │                   │                    │                      │          │
          │           Charge complete       Start loading                 │          │
          │                   │                    │                      │          │
          │                   ▼                    ▼                      │          │
          │           ┌───────────────┐   ┌─────────────────┐             │          │
          │           │   _charged    │   │ _waiting_to_load│             │          │
          │           └───────┬───────┘   └────────┬────────┘             │          │
          │                   │                    │                      │          │
          │           Release or                 Load ammo               │          │
          │           Overload                     │                      │          │
          │                   │                    ▼                      │          │
          │                   │           ┌─────────────────┐             │          │
          │                   │           │_finishing_reload│─────────────┘          │
          │                   │           └─────────────────┘                        │
          │                   │                                                      │
          │                   ▼                                                      │
          │           ┌───────────────┐                                              │
          │           │   _firing     │──────────────┐                               │
          │           └───────────────┘              │                               │
          │                                    Animation done                        │
          │                                          │                               │
          │                                          ▼                               │
          │                                  ┌───────────────┐                       │
          │                                  │  _recovering  │───────────────────────┘
          │                                  └───────────────┘
          │
          ▼
  ┌───────────────┐
  │   _raising    │  (when new weapon selected)
  └───────┬───────┘
          │
          └──────────────────────────────────────────────────────────────────────────┘

Special states for dual-wielded weapons (pistols, shotguns):
  _lowering_for_twofisted_reload    - One hand lowers so other can reload
  _awaiting_twofisted_reload        - Waiting for other hand to lower
  _waiting_for_twofist_to_reload    - Offscreen, waiting for partner
  _sliding_over_to_second_position  - Pistol moving to akimbo position
  _sliding_over_from_second_position - Returning to center
  _waiting_for_other_idle_to_reload  - Waiting for partner to be idle
```

**Firing Pipeline**:
1. Input check (trigger pressed)
2. Ammo validation
3. Rate of fire check (ticks_per_round)
4. Projectile spawn via `new_projectile()`
5. Effects (shell casings, sounds, light)
6. Recovery wait
7. Reload after magazine empty

#### Shell Casing System

Shell casings are first-person visual effects that spawn when certain weapons fire. They're purely cosmetic bullet/cartridge cases that eject from the weapon and fall off-screen.

**Source**: `weapons.c:3670-3787`, `weapon_definitions.h:80-145`

**Shell Casing Types** (`weapon_definitions.h:82-91`):

| ID | Enum Constant | Description | Used By |
|----|---------------|-------------|---------|
| 0 | `_shell_casing_assault_rifle` | MA-75B brass | Assault Rifle |
| 1 | `_shell_casing_pistol` | Magnum brass (center) | Pistol (single) |
| 2 | `_shell_casing_pistol_left` | Magnum brass (left hand) | Dual Pistols |
| 3 | `_shell_casing_pistol_right` | Magnum brass (right hand) | Dual Pistols |
| 4 | `_shell_casing_smg` | SMG brass | KKV-7 SMG |

**Data Structures**:

```c
// From weapon_definitions.h:93-100
struct shell_casing_definition {
    short collection, shape;     // Graphics source (_collection_weapons_in_hand)
    fixed x0, y0;               // Initial position (fixed-point, 0-FIXED_ONE screen coords)
    fixed vx0, vy0;             // Initial velocity (fixed-point units per tick)
    fixed dvx, dvy;             // Velocity delta per tick (gravity/friction)
};

// From weapons.c:120-129 - Runtime state per casing
struct shell_casing_data {
    short type;                 // Shell casing type enum
    short frame;                // Current animation frame
    word flags;                 // _shell_casing_is_reversed for left-hand
    fixed x, y;                 // Current screen position
    fixed vx, vy;               // Current velocity
};

// From weapons.c:131-136 - Per-player storage
struct player_weapon_data {
    // ... weapon state ...
    struct shell_casing_data shell_casings[MAXIMUM_SHELL_CASINGS]; // Max 4 active
};
```

**How Shell Casings Work**:

1. **Spawning** (`new_shell_casing()`, weapons.c:3686):
   - Called when weapon fires (from fire_weapon trigger handlers)
   - Finds free slot in player's shell_casings array (max 4)
   - Copies initial position/velocity from shell_casing_definition
   - Adds randomization: random starting frame, position jitter
   - If reversed flag set (left-hand weapon), negates X velocity

2. **Physics Update** (`update_shell_casings()`, weapons.c:3722):
   - Called every tick from `update_player_weapons()`
   - For each active casing:
     ```c
     shell_casing->x += shell_casing->vx;  // Move horizontally
     shell_casing->y += shell_casing->vy;  // Move vertically
     shell_casing->vx += definition->dvx;  // Apply horizontal drag
     shell_casing->vy += definition->dvy;  // Apply gravity (negative = up!)
     ```
   - Removal condition: `x >= FIXED_ONE` or `x < 0` (off screen sides)

3. **Rendering** (`get_shell_casing_display_data()`, weapons.c:3749):
   - Called by weapon rendering system when building display list
   - Returns weapon_display_information for each active casing
   - Advances animation frame each call
   - Position converted: `vertical_position = FIXED_ONE - y` (screen Y is inverted)

**Actual Definition Values** (weapon_definitions.h:103-144):

```c
// Assault Rifle: Fast horizontal eject, moderate arc
{ _collection_weapons_in_hand, _assault_rifle_shell_casing,
  FIXED_ONE/2 + FIXED_ONE/6, FIXED_ONE/8,   // Start right-center, low
  FIXED_ONE/8, FIXED_ONE/32,                 // Fast right, slow up
  0, -FIXED_ONE/256 }                        // No drag, gravity pulls down

// Pistol Center: Slow eject, longer hang time
{ _collection_weapons_in_hand, _pistol_shell_casing,
  FIXED_ONE/2 + FIXED_ONE/8, FIXED_ONE/4,   // Start right of center, higher
  FIXED_ONE/16, FIXED_ONE/32,                // Slow right, slow up
  0, -FIXED_ONE/400 }                        // Weaker gravity (floats longer)

// Pistol Left/Right: Same physics, different start X and direction
```

**Key Constants**:
- `MAXIMUM_SHELL_CASINGS = 4` - Max active per player at once
- `_shell_casing_is_reversed = 0x0001` - Flip X velocity for left hand
- Position coordinates: 0 = left/top, FIXED_ONE = right/bottom

**Sound Integration**:
- `_snd_assault_rifle_shell_casings` plays when casings spawn
- `_weapon_plays_instant_shell_casing_sound` flag triggers immediate sound

**Which Weapons Use Shell Casings**:
- Assault Rifle: Yes (type 0)
- Pistol: Yes (types 1-3 depending on dual-wield state)
- SMG: Yes (type 4)
- Shotgun: No (shells eject during reload animation instead)
- Fusion Pistol: No (energy weapon)
- Flamethrower: No (continuous stream)
- Alien Weapon: No (energy weapon)
- Rocket Launcher: No (no casings)

### Projectile System

**39 Projectile Types** (`projectiles.h:13-53`):

| ID | Enum Constant | Description | Source |
|----|---------------|-------------|--------|
| 0 | `_projectile_rocket` | SPNKR rocket | Player |
| 1 | `_projectile_grenade` | MA-75B grenade | Player |
| 2 | `_projectile_pistol_bullet` | Magnum round | Player |
| 3 | `_projectile_rifle_bullet` | MA-75B round | Player |
| 4 | `_projectile_shotgun_bullet` | Shotgun pellet | Player |
| 5 | `_projectile_staff` | Staff melee | Pfhor |
| 6 | `_projectile_staff_bolt` | Staff ranged | Pfhor |
| 7 | `_projectile_flamethrower_burst` | Flame | Player |
| 8 | `_projectile_compiler_bolt_minor` | Minor compiler | Compiler |
| 9 | `_projectile_compiler_bolt_major` | Major compiler | Compiler |
| 10 | `_projectile_alien_weapon` | Alien shotgun | Player/Pfhor |
| 11 | `_projectile_fusion_bolt_minor` | Fusion tap | Player |
| 12 | `_projectile_fusion_bolt_major` | Fusion charged | Player |
| 13 | `_projectile_hunter` | Hunter bolt | Hunter |
| 14 | `_projectile_fist` | Melee punch | Player |
| 15 | `_projectile_unused` | (Reserved) | - |
| 16 | `_projectile_armageddon_electricity` | Special | Cyborg |
| 17 | `_projectile_juggernaut_rocket` | Juggernaut missile | Juggernaut |
| 18 | `_projectile_trooper_bullet` | Trooper rifle | Trooper |
| 19 | `_projectile_trooper_grenade` | Trooper grenade | Trooper |
| 20 | `_projectile_minor_defender` | Defender attack | Defender |
| 21 | `_projectile_major_defender` | Defender attack | Defender |
| 22 | `_projectile_juggernaut_missile` | Juggernaut rocket | Juggernaut |
| 23 | `_projectile_minor_energy_drain` | S'pht drain | S'pht |
| 24 | `_projectile_major_energy_drain` | S'pht drain | S'pht |
| 25 | `_projectile_oxygen_drain` | Oxygen damage | Environment |
| 26 | `_projectile_minor_hummer` | Hummer attack | Hummer |
| 27 | `_projectile_major_hummer` | Hummer attack | Hummer |
| 28 | `_projectile_durandal_hummer` | Special hummer | Hummer |
| 29 | `_projectile_minor_cyborg_ball` | Cyborg ball | Cyborg |
| 30 | `_projectile_major_cyborg_ball` | Cyborg ball | Cyborg |
| 31 | `_projectile_ball` | Game ball | Multiplayer |
| 32 | `_projectile_minor_fusion_dispersal` | Fusion shrapnel | Effect |
| 33 | `_projectile_major_fusion_dispersal` | Fusion shrapnel | Effect |
| 34 | `_projectile_overloaded_fusion_dispersal` | Fusion explosion | Effect |
| 35 | `_projectile_yeti` | Yeti attack | Yeti |
| 36 | `_projectile_sewage_yeti` | Sewage yeti | Sewage Yeti |
| 37 | `_projectile_lava_yeti` | Lava yeti | Lava Yeti |
| 38 | `_projectile_smg_bullet` | SMG round | Player |

**Projectile Definition** [projectile_definitions.h:36] (54 bytes):
```c
struct projectile_definition {
    short collection, shape;
    short detonation_effect, media_detonation_effect;
    short contrail_effect;
    short ticks_between_contrails;
    
    world_distance radius;         // Hit radius
    world_distance area_of_effect; // Damage radius
    struct damage_definition damage;
    
    unsigned long flags;
    
    world_distance speed;
    world_distance maximum_range;
};
```

**Key Flags**:
- `_guided` - Homing
- `_affected_by_gravity` - Falls
- `_persistent` - Doesn't vanish
- `_rebounds_from_floor` - Bounces
- `_penetrates_media` - Water pass-through
- `_horizontal_wander` / `_vertical_wander` - Spread

**Damage Types** (from `map.h`):

| Type | ID | Source | Notes |
|------|-----|--------|-------|
| `_damage_explosion` | 0 | Rockets, grenades | Area effect |
| `_damage_electrical_staff` | 1 | Staff weapon | Alien |
| `_damage_projectile` | 2 | Bullets | Standard |
| `_damage_absorbed` | 3 | Shield absorbed | No effect |
| `_damage_flame` | 4 | Flamethrower | Continuous |
| `_damage_hound_claws` | 5 | Hound attack | Melee |
| `_damage_alien_projectile` | 6 | Alien shots | Various |
| `_damage_hulk_slap` | 7 | Hulk attack | Melee |
| `_damage_compiler_bolt` | 8 | Compiler attack | Energy |
| `_damage_fusion_bolt` | 9 | Fusion pistol | Energy |
| `_damage_hunter_bolt` | 10 | Hunter shot | Energy |
| `_damage_fist` | 11 | Punch | Melee |
| `_damage_teleporter` | 12 | Telefrag | Instant kill |
| `_damage_defender` | 13 | Defender attack | Energy |
| `_damage_yeti_claws` | 14 | Yeti attack | Melee |
| `_damage_yeti_projectile` | 15 | Yeti shot | Projectile |
| `_damage_crushing` | 16 | Platform/door | Environmental |
| `_damage_lava` | 17 | Lava contact | Environmental |
| `_damage_suffocation` | 18 | No oxygen | Environmental |
| `_damage_goo` | 19 | Sewage/goo | Environmental |
| `_damage_energy_drain` | 20 | Shield drain | Special |
| `_damage_oxygen_drain` | 21 | O2 drain | Special |
| `_damage_hummer_bolt` | 22 | Hummer attack | Energy |
| `_damage_shotgun_projectile` | 23 | Shotgun | Multi-hit |

**Monster Immunities/Weaknesses**: Stored as bitmasks using `FLAG(_damage_type)`

### Effects System

**85+ Effect Types**:

- Impact: Bullet ricochet, blood, sparks
- Environmental: Water/lava splashes (3 sizes each)
- Weapon: Rocket contrails, shell casings
- Teleport: In/out effects
- Death: Faction-specific

**Effect Data** [effects.h:90] (16 bytes):
```c
struct effect_data {
    short type;
    short object_index;    // Sprite
    word flags;
    short data;            // Special data
    short delay;           // Visibility delay
};
```

**Lifecycle**:
1. Spawn via `new_effect()`
2. Delay (invisible ticks)
3. Animation play
4. Sound trigger
5. Auto-removal after animation

Maximum: 64 simultaneous effects.

### Motion Sensor System

The motion sensor (radar) displays nearby entities on a circular HUD element.

**Core Constants** (from `motion_sensor.c`):
```c
#define MAXIMUM_MOTION_SENSOR_ENTITIES 12
#define NUMBER_OF_PREVIOUS_LOCATIONS 6      // Trail effect
#define MOTION_SENSOR_UPDATE_FREQUENCY 5    // Ticks between updates
#define MOTION_SENSOR_RESCAN_FREQUENCY 15   // Ticks between full rescans
#define MOTION_SENSOR_RANGE (8*WORLD_ONE)   // Detection radius
#define MOTION_SENSOR_SCALE 7               // World-to-screen scale
#define FLICKER_FREQUENCY 0xf               // Magnetic interference
```

**Entity Data Structure**:
```c
struct entity_data {
    word flags;                 // [slot_used.1] [being_removed.1] [unused.14]
    short monster_index;
    shape_descriptor shape;     // Blip appearance
    short remove_delay;         // Fade-out counter [0, NUMBER_OF_PREVIOUS_LOCATIONS)
    point2d previous_points[NUMBER_OF_PREVIOUS_LOCATIONS];  // Trail history
    boolean visible_flags[NUMBER_OF_PREVIOUS_LOCATIONS];
    world_point3d last_location;
    angle last_facing;
};
```

**Motion Sensor Pipeline**:

```
motion_sensor_scan() called every tick:
      │
      ├─► ticks_since_last_update < UPDATE_FREQUENCY?
      │           │
      │           └─► YES: Return (no update)
      │
      ├─► Reset update counter
      │
      ├─► ticks_since_last_rescan >= RESCAN_FREQUENCY?
      │           │
      │           └─► YES: Scan for new entities in range
      │                   │
      │                   ├─► For each monster in range:
      │                   │     └─► find_or_add_motion_sensor_entity()
      │                   │
      │                   └─► Mark out-of-range entities for removal
      │
      ├─► Update entity positions
      │     └─► Store in previous_points[] ring buffer
      │
      └─► Set motion_sensor_changed = TRUE
```

**Blip Types**:
| Shape | Source | Meaning |
|-------|--------|---------|
| `alien_shapes` | Hostile aliens | Danger |
| `friendly_shapes` | Allied units | Friendly |
| `enemy_shapes` | Hostile players | Enemy (multiplayer) |
| `compass_shapes` | Network game | Team compass |

**Trail Effect**:
```
Blip movement shown as fading trail:

    Current position (brightest)
              ●
             ◐
            ◔
           ◌         Previous positions (fading)
          ○
         ·           Oldest position (faintest)

NUMBER_OF_PREVIOUS_LOCATIONS = 6 positions tracked
```

**Removal Animation**:
When entity leaves range, it's marked `SLOT_IS_BEING_REMOVED` and fades out over `remove_delay` ticks before slot is freed.

### Terminal System (Computer Interface)

Terminals provide story content through an interactive text/image system.

**Terminal Commands** (parsed during preprocessing):

| Command | Syntax | Description |
|---------|--------|-------------|
| `#LOGON` | `#LOGON XXXX` | Login screen (XXXX = shape) |
| `#UNFINISHED` | `#UNFINISHED` | Unfinished mission text |
| `#SUCCESS` | `#SUCCESS` | Success mission text |
| `#FAILURE` | `#FAILURE` | Failure mission text |
| `#INFORMATION` | `#INFORMATION` | General information |
| `#CHECKPOINT` | `#CHECKPOINT XX` | Goal checkpoint (XX = goal) |
| `#SOUND` | `#SOUND XXXX` | Play sound effect |
| `#MOVIE` | `#MOVIE XXXX` | Play movie (from Movie file) |
| `#TRACK` | `#TRACK XXXX` | Play music (from Music file) |
| `#PICT` | `#PICT XXXX` | Display PICT image |
| `#INTERLEVEL TELEPORT` | `#INTERLEVEL TELEPORT XXX` | Go to level XXX |
| `#INTRALEVEL TELEPORT` | `#INTRALEVEL TELEPORT XXX` | Go to polygon XXX |
| `#END` | `#END` | End current group |

**Text Formatting Codes**:
| Code | Effect |
|------|--------|
| `$B` | Bold on |
| `$b` | Bold off |
| `$I` | Italic on |
| `$i` | Italic off |
| `$U` | Underline on |
| `$u` | Underline off |

**Preprocessed Terminal Structure**:
```c
struct static_preprocessed_terminal_data {
    short total_length;
    short flags;
    short lines_per_page;      // For internationalization
    short grouping_count;
    short font_changes_count;
    // Followed by:
    // struct terminal_groupings groups[grouping_count];
    // struct text_face_data[font_changes_count];
    // char text[];
};
```

**Terminal State Machine**:
```
Player approaches terminal
           │
           ▼
    ┌─────────────┐
    │ enter_computer│
    │ _interface() │
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐     Tab/Space      ┌─────────────┐
    │ Display page ├──────────────────►│ Next page   │
    └──────┬──────┘                    └──────┬──────┘
           │                                  │
           │ Last page                        │
           ▼                                  │
    ┌─────────────┐◄──────────────────────────┘
    │ Check ending│
    │   command   │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │             │
Teleport?     Exit
    │             │
    ▼             ▼
goto_level() abort_terminal_mode()
```

### Replay/Recording System

Marathon supports recording and playback of gameplay through action queue capture.

**Core Constants** (from `vbl.h`):
```c
#define RECORD_CHUNK_SIZE (MAXIMUM_QUEUE_SIZE/2)
#define MAXIMUM_TIME_DIFFERENCE 15        // Ticks tolerance
#define MAXIMUM_NET_QUEUE_SIZE 8
#define MAXIMUM_REPLAY_SPEED 5
#define MINIMUM_REPLAY_SPEED (-5)
```

**Action Queue Structure**:
```c
struct action_queue {
    short read_index, write_index;
    long* buffer;               // Circular buffer of action flags
};
```

**Recording Process**:
```
Game Loop (recording):
       │
       ├─► Collect local input → action_flags
       │
       ├─► Store in action_queue
       │
       ├─► Every RECORD_CHUNK_SIZE actions:
       │     └─► Write chunk to replay file
       │
       └─► Continue until game ends
```

**Replay File Format**:
```
[Replay Header]
  - game_data
  - player_start_data
  - entry_point
  - random_seed
[Action Chunk 0]
  - action_flags[RECORD_CHUNK_SIZE]
[Action Chunk 1]
  - action_flags[RECORD_CHUNK_SIZE]
...
[EOF]
```

**Playback Modes**:
| Speed | Effect |
|-------|--------|
| `MINIMUM_REPLAY_SPEED (-5)` | Slowest playback |
| `-1` | Half speed |
| `0` | Normal speed (1x) |
| `1` | 2x speed |
| `MAXIMUM_REPLAY_SPEED (5)` | Fastest playback |
| `toggle_ludicrous_speed()` | Skip rendering (fastest) |

**Playback Synchronization**:
```
replay_interpolate_world_into_camera():
    │
    ├─► Check ticks_since_last_update
    │
    ├─► If behind: Run extra game ticks (catch up)
    │
    ├─► If ahead: Wait (throttle playback)
    │
    └─► Maximum drift: MAXIMUM_TIME_DIFFERENCE ticks
```

### Save/Load System

Games are saved as Marathon WAD files containing complete world state.

**Save Game Tags** (from `game_wad.c`):

| Tag | Contents | Loaded from level? |
|-----|----------|-------------------|
| `ENDPOINT_DATA_TAG` | Vertex data | Yes |
| `LINE_TAG` | Line definitions | Yes |
| `SIDE_TAG` | Wall textures | Yes |
| `POLYGON_TAG` | Room data | Yes |
| `LIGHTSOURCE_TAG` | Light states | No |
| `OBJECT_TAG` | Map objects | Yes |
| `MAP_INFO_TAG` | Level metadata | Yes |
| `MEDIA_TAG` | Liquid states | No |
| `PLAYER_STRUCTURE_TAG` | Player data | No |
| `DYNAMIC_STRUCTURE_TAG` | World state | No |
| `OBJECT_STRUCTURE_TAG` | Object instances | No |
| `MONSTERS_STRUCTURE_TAG` | Monster states | No |
| `EFFECTS_STRUCTURE_TAG` | Active effects | No |
| `PROJECTILES_STRUCTURE_TAG` | Active projectiles | No |
| `PLATFORM_STRUCTURE_TAG` | Platform states | No |
| `WEAPON_STATE_TAG` | Weapon states | No |
| `TERMINAL_STATE_TAG` | Terminal progress | No |
| `AUTOMAP_LINES` | Explored lines | No |
| `AUTOMAP_POLYGONS` | Explored areas | No |

**Revert Game System**:
```c
struct revert_game_info {
    boolean game_is_from_disk;          // TRUE = loaded, FALSE = new game
    struct game_data game_information;
    struct player_start_data player_start;
    struct entry_point entry_point;
    FileDesc saved_game;                // File reference
};
```

**Save/Load Pipeline**:

```
save_game_file():
    │
    ├─► build_save_game_wad()
    │     ├─► Pack all tag data
    │     ├─► Include dynamic state (monsters, projectiles, etc.)
    │     └─► Calculate checksums
    │
    ├─► Write WAD header
    │
    ├─► Write WAD data
    │
    └─► Update revert_game_data (for quick-load)

load_game_from_file():
    │
    ├─► Open WAD file
    │
    ├─► read_wad_header()
    │
    ├─► read_indexed_wad_from_file()
    │
    ├─► process_map_wad()
    │     ├─► Restore static geometry
    │     ├─► Restore dynamic state
    │     └─► Restore terminal progress
    │
    └─► Update revert_game_data
```

**Key Save Functions**:
- `save_game_file()` - Write current state to WAD
- `load_game_from_file()` - Restore state from WAD
- `revert_game()` - Quick-reload last save point
- `setup_revert_game_info()` - Configure revert state

**Level vs Save Difference**:
- **Level load**: Only loads "loaded_by_level = TRUE" tags
- **Save load**: Loads ALL tags (complete state restoration)

This distinction ensures saved games restore exact state (monster positions, health, etc.) while level loads start fresh.

---

## 9. Networking Architecture

> **🔧 For Porting:** For single-player, stub out all networking (return from functions early). For multiplayer, replace `network_ddp.c` and `network_adsp.c` with modern sockets/UDP. The core sync logic in `network.c` and `network_games.c` is portable—it just needs a transport layer.

### Deterministic Peer-to-Peer

Marathon pioneered deterministic networking for FPS games, using a radically different approach from modern client-server architectures.

**Key Principles**:
- Send **inputs**, not **state**
- Each machine runs identical simulation
- Same inputs → same outputs (fixed-point math)
- Validation via checksums

#### How Deterministic Networking Works

**The Core Concept**:

```
Traditional Networking (Client-Server):
    Client 1 ──[Position, Actions]──> Server ──[World State]──> Client 1
    Client 2 ──[Position, Actions]──> Server ──[World State]──> Client 2
    Client 3 ──[Position, Actions]──> Server ──[World State]──> Client 3

    Problem: High bandwidth (sending entire world state)
    Advantage: Server is authoritative

Marathon's Deterministic Networking (Peer-to-Peer):
    Peer 1 ──[Input Flags]──> All Peers
    Peer 2 ──[Input Flags]──> All Peers
    Peer 3 ──[Input Flags]──> All Peers

    Each peer:
    1. Collects all inputs for tick N
    2. Runs IDENTICAL simulation with same inputs
    3. Results in IDENTICAL world state (determinism)

    Advantage: Low bandwidth (only inputs)
    Requirement: Bit-identical simulation on all machines
```

**Why This Works**:

```
Fixed-Point Mathematics Ensures Determinism:

    Floating Point (NON-deterministic):
        Machine A: 1.0 / 3.0 = 0.33333333333...  (rounded differently)
        Machine B: 1.0 / 3.0 = 0.33333334000...  (slightly different)
        After 1000 operations → positions diverge!

    Fixed-Point (DETERMINISTIC):
        Machine A: 65536 / 3 = 21845  (exact integer division)
        Machine B: 65536 / 3 = 21845  (same result)
        After 1000 operations → positions identical!

    Marathon uses:
        FIXED_ONE = 65536 (16.16 fixed-point)
        All physics calculations use integer math
        Same input + same algorithm = same output
```

#### Peer-to-Peer Architecture

**Network Topology**:

```
4-Player Game (Full Mesh):

    Player 1                    Player 2
       ●─────────────────────────●
       │╲                       ╱│
       │ ╲                     ╱ │
       │  ╲                   ╱  │
       │   ╲                 ╱   │
       │    ╲               ╱    │
       │     ╲             ╱     │
       │      ╲           ╱      │
       │       ╲         ╱       │
       │        ╲       ╱        │
       │         ╲     ╱         │
       ●───────────●───────────●
    Player 4                    Player 3

Each player broadcasts to all others
No central server
All peers are equal
Maximum 8 players (n² connections)
```

**Action Queue System**:
```c
struct action_flags {
    long flags;            // Compressed input (32 bits)
    short tick;            // Which tick
    short player_index;
};
```

#### Network Synchronization Loop

**Per-Frame Process**:

```
Tick N begins:

Player 1's Machine:
    ┌─────────────────────────────────────┐
    │ 1. Read local input (keyboard/mouse)│
    │    → Generate action_flags           │
    │                                      │
    │ 2. Broadcast to all peers:           │
    │    send(action_flags, tick=N, p=1)   │
    │                                      │
    │ 3. Wait for all other players:       │
    │    recv(action_flags, tick=N, p=2)   │
    │    recv(action_flags, tick=N, p=3)   │
    │    recv(action_flags, tick=N, p=4)   │
    │                                      │
    │ 4. ALL inputs for tick N ready!      │
    │                                      │
    │ 5. Run game logic for 1 tick:        │
    │    update_players(all_actions)       │
    │    move_projectiles()                │
    │    move_monsters()                   │
    │    physics_update()                  │
    │                                      │
    │ 6. World state now at tick N+1       │
    │                                      │
    │ 7. Render current state              │
    └─────────────────────────────────────┘

Player 2's Machine:
    [IDENTICAL PROCESS]
    Same inputs → Same simulation → Same result

Player 3's Machine:
    [IDENTICAL PROCESS]

Player 4's Machine:
    [IDENTICAL PROCESS]
```

**Timeline Diagram**:

```
Time →
Tick:      N              N+1            N+2            N+3

Player 1: [Input] [Wait] [Sim] [Render] [Input] [Wait] [Sim]...
Player 2: [Input] [Wait] [Sim] [Render] [Input] [Wait] [Sim]...
Player 3: [Input] [Wait] [Sim] [Render] [Input] [Wait] [Sim]...
Player 4: [Input] [Wait] [Sim] [Render] [Input] [Wait] [Sim]...

[Wait] = Waiting for all players' inputs
[Sim]  = Running deterministic simulation
[Render] = Drawing frame (can be faster than 30Hz)

All machines stay synchronized at tick boundaries
```

#### Lag Tolerance and Queue Buffering

**Input Queue System**:

```
Each player maintains a queue of future actions:

Current Tick: 100

Action Queue (buffered ahead):
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│ 98  │ 99  │ 100 │ 101 │ 102 │ 103 │ 104 │ ← Tick numbers
└─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  Past  Past  Now!  Buffered future actions

Buffer Size: 1-2 seconds (30-60 ticks)

If late packet arrives:
    - Tick 101 action arrives while we're at tick 100
    - Insert into queue at correct position
    - Simulation continues normally

If very late packet arrives:
    - Tick 99 action arrives while we're at tick 100
    - Already simulated!
    - DESYNC DETECTED!
    - Game may pause or disconnect
```

**Handling Network Delays**:

```
Scenario: Player 3 has high ping

Tick 50:
    Player 1: Send input for tick 50 ─────┐
    Player 2: Send input for tick 50 ─────┼───> All arrive at tick 50
    Player 3: Send input for tick 50 ─────┘ (delayed)
              ↓
              ↓ (traveling over network)
              ↓
Tick 51:      ↓
    Player 1, 2: WAITING for Player 3's tick 50 input
    Player 3's packet arrives!

    All machines now have inputs for tick 50
    Simulation proceeds

Game feels "laggy" but stays synchronized
All players see the same world
```

#### Determinism Requirements

**Why Fixed Timestep is Critical**:

```
Variable Timestep (BREAKS determinism):
    Machine A: Fast CPU
        Tick 1: dt = 0.016s → physics_update(0.016)
        Tick 2: dt = 0.017s → physics_update(0.017)

    Machine B: Slow CPU
        Tick 1: dt = 0.033s → physics_update(0.033)
        Tick 2: dt = 0.034s → physics_update(0.034)

    Result: Different physics! Desync!

Fixed Timestep (ENSURES determinism):
    All Machines:
        Tick 1: physics_update(1/30)  ← Always exactly 1/30 second
        Tick 2: physics_update(1/30)
        Tick 3: physics_update(1/30)

    Result: Identical physics! Stays in sync!
```

**Random Number Generation**:

```
Deterministic Random (Marathon's approach):

    Seed synchronized at game start:
        All players: random_seed = 12345

    During gameplay:
        Player 1: random() → 8472  (using seed 12345)
        Player 2: random() → 8472  (using same seed 12345)
        Player 3: random() → 8472  (identical!)

        Next call:
        Player 1: random() → 2391
        Player 2: random() → 2391
        Player 3: random() → 2391

    All random events (damage, spread, etc.) are identical!

Non-deterministic Random (would break):
    Player 1: random() → based on CPU timer → 1234
    Player 2: random() → based on CPU timer → 8876
    Player 3: random() → based on CPU timer → 4429

    Desync immediately!
```

**Desync Prevention**:
- Fixed-point math (no floats)
- Deterministic random seed
- Position checksum validation
- Bit-identical simulation across platforms

#### Action Flags Compression

**Data Compression**:
32-bit action flags encode all player input:

```
Bit Layout (32 bits total):

Bits 0-6:   Absolute yaw (7 bits)        → 128 angles
Bits 7-13:  Reserved/flags (7 bits)
Bits 14-18: Absolute pitch (5 bits)      → 32 angles
Bits 19-21: Reserved/flags (3 bits)
Bits 22-28: Absolute position (7 bits)   → Position encoding
Bits 29-31: Movement flags (3 bits)

Additional flags:
_turning_left, _turning_right
_looking_left, _looking_right, _looking_up, _looking_down, _looking_center
_moving_forward, _moving_backward
_sidestepping_left, _sidestepping_right
_run_dont_walk
_left_trigger_state, _right_trigger_state
_action_trigger_state
_cycle_weapons_forward, _cycle_weapons_backward
_toggle_map, _swim

Result: Entire player input in just 32 bits = 4 bytes per player per tick

Bandwidth: 4 players × 4 bytes × 30 ticks/sec = 480 bytes/sec
           (Plus overhead, ~1-2 KB/sec total)

Compare to modern games: 10-100 KB/sec per player!
```

#### Checksum Validation

**Preventing Desyncs**:

```
Periodic Position Checksum (every 10 ticks):

Tick 100:
    Player 1 calculates: CRC32(all_player_positions) = 0xABCD1234
    Player 1 sends: checksum = 0xABCD1234

Tick 100:
    Player 2 calculates: CRC32(all_player_positions) = 0xABCD1234
    Player 2 sends: checksum = 0xABCD1234

Tick 100:
    Player 3 calculates: CRC32(all_player_positions) = 0xFFFF9999  ← Different!
    Player 3 sends: checksum = 0xFFFF9999

Tick 101:
    All players receive checksums
    Player 1: "My checksum matches Player 2, but not Player 3"
    Player 2: "My checksum matches Player 1, but not Player 3"
    Player 3: "I don't match anyone!"

    DESYNC DETECTED!

    Options:
    - Pause game and warn players
    - Disconnect desynced player
    - Attempt resynchronization
```

**Why Desyncs Happen**:
```
Common Causes:
1. Floating-point math used somewhere (breaks determinism)
2. Uninitialized memory read (random values)
3. Platform-specific behavior (different compilers)
4. Race conditions in code
5. Save/load corrupting state

Marathon's Solutions:
1. ✓ No floating-point math (all fixed-point)
2. ✓ Careful initialization
3. ✓ Same codebase on all platforms
4. ✓ Single-threaded simulation
5. ✓ Checksums verify state consistency
```

#### Network Loop Implementation

**Network Loop**:
1. Gather local input → action flags
2. Broadcast to all peers
3. Wait for all players' inputs for tick N
4. Advance world by 1 tick with all inputs
5. Repeat

**Actual Code Flow**:

```
High-level pseudocode:

void network_game_loop() {
    while (game_running) {
        // 1. Get local player input
        action_flags local_input = get_player_input();
        local_input.tick = current_tick;
        local_input.player_index = local_player_index;

        // 2. Broadcast to all peers
        for (each peer) {
            send_packet(peer, &local_input);
        }

        // 3. Wait for all peer inputs
        action_flags all_inputs[MAX_PLAYERS];
        all_inputs[local_player_index] = local_input;

        for (each remote player) {
            receive_packet(remote_player, &all_inputs[remote_player]);

            // Verify tick number matches
            if (all_inputs[remote_player].tick != current_tick) {
                // Handle late/early packet
                handle_timing_issue();
            }
        }

        // 4. ALL inputs collected, advance simulation
        update_world_one_tick(all_inputs);

        // 5. Increment tick counter
        current_tick++;

        // 6. Render current state (can be faster than 30Hz)
        render_frame();
    }
}
```

**Lag Tolerance**:
- Queue buffers 1-2 seconds
- Late arrivals caught up
- Monster AI waits for slowest player

---

## 10. File Formats

> **🔧 For Porting:** Great news! All game data files (Maps, Shapes, Sounds) are readable with standard `fopen()`/`fread()`. Key changes:
> - Replace `FSSpec`/`FSRead` with stdio in `wad.c`, `game_wad.c`
> - Add byte swapping (files are big-endian, x86 is little-endian)
> - Replace `BlockMove()` with `memcpy()`
> - Only the optional Images file uses Mac resource forks (can stub or pre-extract)

### Marathon WAD Format (Maps Only)

Custom format, NOT Doom WADs.

**WAD File Layout**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    WAD FILE STRUCTURE                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │            WAD HEADER (128 bytes)                         │  │
│  │  ├─ version, data_version                                 │  │
│  │  ├─ file_name[64]                                         │  │
│  │  ├─ checksum (CRC32)                                      │  │
│  │  ├─ directory_offset ─────────────────────────────┐       │  │
│  │  ├─ wad_count (number of levels)                  │       │  │
│  │  └─ parent_checksum (for patches)                 │       │  │
│  └───────────────────────────────────────────────────│───────┘  │
│                                                      │          │
│  ┌───────────────────────────────────────────────────│───────┐  │
│  │         LEVEL DATA (repeated per level)           │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='PNTS' length=N              │       │  │
│  │  [Point Data - N bytes]                          │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='LINS' length=M              │       │  │
│  │  [Line Data - M bytes]                           │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='SIDS' length=...            │       │  │
│  │  [Side Data]                                     │       │  │
│  │                                                   │       │  │
│  │  [Entry Header] tag='POLY' length=...            │       │  │
│  │  [Polygon Data]                                  │       │  │
│  │                                                   │       │  │
│  │  ... more tagged entries ...                     │       │  │
│  └───────────────────────────────────────────────────│───────┘  │
│                                                      │          │
│  ┌───────────────────────────────────────────────────▼───────┐  │
│  │         DIRECTORY (at directory_offset)                   │  │
│  │                                                           │  │
│  │  [Directory Entry 0] offset, length, index=0              │  │
│  │  [Directory Entry 1] offset, length, index=1              │  │
│  │  ... one per level ...                                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**WAD Header** (128 bytes):
```c
struct wad_header {
    short version;                           // 0=pre-entry, 2=M2, 4=Infinity
    short data_version;                      // Application-defined
    char file_name[64];                      // Internal filename
    unsigned long checksum;                  // CRC32 of file
    long directory_offset;                   // Offset to directory
    short wad_count;                         // Number of entries (levels)
    short application_specific_directory_data_size;  // Extra dir data
    short entry_header_size;                 // 16 bytes
    short directory_entry_base_size;         // 10 bytes
    unsigned long parent_checksum;           // Non-zero = patch file
    short unused[20];                        // Reserved
};
```

**Directory Entry** (10 bytes):
```c
struct directory_entry {
    long offset_to_start;
    long length;
    short index;
};
```

**WAD Data Structure**:
```
[Entry Header - 16 bytes]
[Data]
[Entry Header - 16 bytes]
[Data]
...
```

**Entry Header** (16 bytes):
```c
struct entry_header {
    long tag;            // 'PNTS', 'LINS', 'SIDS', 'POLY', etc.
    long next_offset;    // Relative offset
    long length;
    long offset;
};
```

**Tag Types**:
- `'PNTS'` - Vertex points
- `'LINS'` - Line segments
- `'SIDS'` - Side definitions
- `'POLY'` - Polygon data
- `'LITE'` - Light sources
- `'OBJS'` - Object placement
- `'Minf'` - Map info
- `'plat'` - Platforms
- `'medi'` - Media (liquids)
- Many more (see tags.h)

### Shape Files (Binary Data Fork)

Despite being a Mac game, shapes are stored in **standard data fork** as binary files.

**Collection Header Table** (at file start):
```c
struct collection_header {
    long offset, length;        // 8-bit collection data
    long offset16, length16;    // 16-bit collection data
    
    // Runtime (not in file):
    struct collection_definition **collection;
    void **shading_tables;
};
```

**MAXIMUM_COLLECTIONS** = 32

**Reading**:
```c
FILE* fp = fopen("Shapes16", "rb");
struct collection_header headers[32];
fread(&headers, sizeof(struct collection_header), 32, fp);

// Load collection #5 at 16-bit
fseek(fp, headers[5].offset16, SEEK_SET);
byte* data = malloc(headers[5].length16);
fread(data, headers[5].length16, 1, fp);
```

**Collection Definition** (512 bytes):
```c
struct collection_definition {
    short version;               // Should be 3
    short type;
    word flags;
    
    short color_count, clut_count;
    long color_table_offset;
    
    short high_level_shape_count;
    long high_level_shape_offset_table_offset;
    
    short low_level_shape_count;
    long low_level_shape_offset_table_offset;
    
    short bitmap_count;
    long bitmap_offset_table_offset;
    
    short pixels_to_world;
    long size;
};
```

**High-Level Shape** - Animation sequence:
```c
struct high_level_shape_definition {
    short type;
    word flags;
    char name[33];
    short number_of_views;       // Angles (1, 3, 5, 8)
    short frames_per_view;
    short ticks_per_frame;
    short key_frame;
    short transfer_mode;
    short first_frame_sound, key_frame_sound, last_frame_sound;
    short pixels_to_world;
    short loop_frame;
    short low_level_shape_indexes[...];  // Views × Frames
};
```

**Low-Level Shape** - Individual frame:
```c
struct low_level_shape_definition {
    word flags;                 // Mirroring, obscured
    fixed minimum_light_intensity;
    short bitmap_index;
    short origin_x, origin_y;
    short key_x, key_y;
    short world_left, world_right, world_top, world_bottom;
    short world_x0, world_y0;
};
```

**Bitmap Data**:
- Raw: Width × Height bytes
- RLE compressed: Scanline-based

**RLE Format**:
- If byte < 128: Copy next `byte` pixels literally
- If byte >= 128: Repeat next pixel `(256 - byte)` times

**Color Tables**:
```c
struct rgb_color_value {
    byte flags;      // SELF_LUMINESCENT = 0x80
    byte value;      // Brightness
    word red;        // 0-65535
    word green;
    word blue;
};
```

**Important**: All data is **big-endian** (Mac byte order). Must byte-swap on x86.

---

## 11. Performance and Optimization

### Portal Culling

**Biggest Win**: Only 50-100 of 1000 polygons typically rendered.

Average scene:
- Total polygons: 500-1000
- Visible after culling: 50-100
- 10× reduction in rendering cost

### Assembly Optimizations

**Critical Inner Loops** (68K and PowerPC versions):
- `_texture_horizontal_polygon_lines8/16/32()`
- `_texture_vertical_polygon_lines8/16/32()`
- Fixed-point multiply/divide

**C Fallbacks**: Exist for all assembly routines.

### Fixed-Point Math

**Performance Benefits**:
- No FPU required
- All integer operations
- Bit shifts instead of division where possible
- Precomputed trig tables

**Trig Tables**:
```c
#define NUMBER_OF_ANGLES 512
fixed sine_table[NUMBER_OF_ANGLES];
fixed cosine_table[NUMBER_OF_ANGLES];
```

Indexed by `angle >> (FIXED_FRACTIONAL_BITS - ANGULAR_BITS)`.

### Precalculation

**Edge Tables**: Bresenham's algorithm run once per polygon, results cached.

**Shading Tables**: Built once at load time:
- 8-bit: 32 tables × 256 entries = 8 KB
- 16-bit: 64 tables × 256 entries × 2 bytes = 32 KB
- 32-bit: 64 tables × 256 entries × 4 bytes = 64 KB

**Neighbor Lists**: Polygons within WORLD_ONE distance precomputed for rapid queries.

### Staggered Updates

**Monster AI**: Only one expensive pathfinding per frame.

**Pathfinding**: Updates once per 4 ticks (120ms).

**Round-robin**: Iterate monster array with fairness.

### Memory Management

**Mac Handles**: Used for large allocations (shape collections, shading tables).

On classic Mac OS, handles allowed memory compaction to avoid fragmentation.

For modern ports: Replace with regular `malloc()`.

---

## 12. Appendix: Data Structures

### Core Types

```c
typedef long fixed;                  // 16.16 fixed-point
typedef short world_distance;        // 10 fractional bits
typedef unsigned short word;
typedef unsigned char byte;
typedef byte boolean;
typedef short angle;                 // 512 angles per circle

struct world_point2d { world_distance x, y; };
struct world_point3d { world_distance x, y, z; };
struct fixed_point3d { fixed x, y, z; };
struct world_vector2d { world_distance i, j; };
struct world_vector3d { world_distance i, j, k; };
struct fixed_vector3d { fixed i, j, k; };
```

### Key Constants

```c
// Fixed-point
#define FIXED_ONE (1<<16)
#define FIXED_FRACTIONAL_BITS 16

// World units
#define WORLD_ONE 1024
#define WORLD_FRACTIONAL_BITS 10

// Timing
#define TICKS_PER_SECOND 30
#define MACHINE_TICKS_PER_SECOND 60

// Angles
#define NUMBER_OF_ANGLES 512
#define FULL_CIRCLE 512
#define HALF_CIRCLE 256
#define QUARTER_CIRCLE 128
#define EIGHTH_CIRCLE 64
#define SIXTEENTH_CIRCLE 32

// Trig
#define TRIG_SHIFT 10
#define TRIG_MAGNITUDE (1<<TRIG_SHIFT)

// Rendering
#define MAXIMUM_NODES 512
#define MAXIMUM_SORTED_NODES 128
#define MAXIMUM_RENDER_OBJECTS 72
#define MAXIMUM_CLIPPING_WINDOWS 256
#define NORMAL_FIELD_OF_VIEW 80

// Physics
#define GRAVITATIONAL_ACCELERATION (FIXED_ONE/400)
#define TERMINAL_VELOCITY (FIXED_ONE/7)
#define COEFFICIENT_OF_ABSORBTION 2
```

### Structure Sizes

| Structure | Size (bytes) | Count |
|-----------|--------------|-------|
| world_point2d | 4 | Variable |
| line_data | 32 | 1024 max |
| side_data | 64 | 2048 max |
| polygon_data | 128 | 512 max |
| platform_data | Variable | 256 max |
| media_data | Variable | 32 max |
| light_data | Variable | 256 max |
| monster_definition | 128 | 47 types |
| weapon_definition | 196 | ~10 types |
| projectile_definition | 54 | ~40 types |
| effect_data | 16 | 64 max |

---

## 13. Sound System

> **🔧 For Porting:** Replace `sound_macintosh.c` entirely with your audio backend (recommend miniaudio.h). Keep `game_sound.c` logic (3D positioning, channel management) and redirect its output calls to your platform layer. Sound data in Sounds files is standard PCM—just need to parse the headers.

Marathon features a sophisticated 3D audio system with spatial positioning, obstruction detection, and dynamic tracking.

### Sound Architecture

**Core Constants**:
```c
#define MAXIMUM_SOUND_CHANNELS 4         // Normal sound channels
#define MAXIMUM_AMBIENT_SOUND_CHANNELS 2 // Background/environmental
#define MAXIMUM_SOUND_VOLUME 256         // Full volume
#define NUMBER_OF_SOUND_VOLUME_LEVELS 8  // User preference levels
#define ABORT_AMPLITUDE_THRESHHOLD (MAXIMUM_SOUND_VOLUME/6)  // ~42
#define MINIMUM_RESTART_TICKS (MACHINE_TICKS_PER_SECOND/12)  // ~5 ticks
```

**Initialization Flags**:
| Flag | Value | Description |
|------|-------|-------------|
| `_stereo_flag` | 0x0001 | Enable stereo panning |
| `_dynamic_tracking_flag` | 0x0002 | Track moving sound sources |
| `_doppler_shift_flag` | 0x0004 | Pitch shift based on velocity |
| `_ambient_sound_flag` | 0x0008 | Enable ambient sounds |
| `_16bit_sound_flag` | 0x0010 | Use 16-bit audio |
| `_more_sounds_flag` | 0x0020 | Load additional sound variations |
| `_extra_memory_flag` | 0x0040 | Use extra memory for sound cache |

### Channel System

**Channel Data Structure**:
```c
struct channel_data {
    word flags;                      // Channel state flags
    short sound_index;               // Currently playing sound
    short identifier;                // Unique sound instance ID
    struct sound_variables variables; // Volume, pitch, etc.
    world_location3d *dynamic_source; // Moving source (tracked)
    world_location3d source;          // Static source position
    unsigned long start_tick;         // When playback started
};
```

**Channel Allocation**:
```
Audio Channel Layout:
┌─────────────────────────────────────────────────────────────┐
│  Channel 0   │  Channel 1   │  Channel 2   │  Channel 3    │
│  (Normal)    │  (Normal)    │  (Normal)    │  (Normal)     │
├─────────────────────────────────────────────────────────────┤
│       Channel 4 (Ambient)   │       Channel 5 (Ambient)    │
└─────────────────────────────────────────────────────────────┘

Priority System:
  1. Higher volume sounds preempt lower volume
  2. Closer sounds preempt distant sounds
  3. Ambient channels never preempt normal channels
  4. ABORT_AMPLITUDE_THRESHHOLD prevents very quiet sounds from interrupting
```

### 3D Sound Positioning

**Distance Attenuation**:
```c
// Volume decreases with distance squared
volume = base_volume * (MAXIMUM_SOUND_DISTANCE - distance) / MAXIMUM_SOUND_DISTANCE;
volume = MAX(volume, 0);  // Clamp to zero
```

**Stereo Panning**:
```c
// Calculate angle from listener to source
angle = arctangent(source.x - listener.x, source.y - listener.y);
relative_angle = NORMALIZE_ANGLE(angle - listener.facing);

// Pan based on relative angle
// 0° = center, 90° = full right, 270° = full left
left_volume = volume * (HALF_CIRCLE - ABS(relative_angle - QUARTER_CIRCLE)) / HALF_CIRCLE;
right_volume = volume * (HALF_CIRCLE - ABS(relative_angle - THREE_QUARTER_CIRCLE)) / HALF_CIRCLE;
```

**Visualization**:
```
                    Source
                       ●
                      /
                     / distance
                    /
                   /
        @─────────+  Listener (facing →)
                  │
                  │ angle = 45°
                  ▼

        Result: Left=70%, Right=100%
```

### Sound Obstruction

**Obstruction Flags**:
| Flag | Value | Effect |
|------|-------|--------|
| `_sound_was_obstructed` | 0x0001 | Wall between source and listener |
| `_sound_was_media_obstructed` | 0x0002 | Liquid surface between source and listener |
| `_sound_was_media_muffled` | 0x0004 | Both in liquid but separated |

**Obstruction Algorithm**:
```
Line-of-Sound Check:
    1. Cast ray from listener to sound source
    2. For each polygon boundary crossed:
       - If solid wall: _sound_was_obstructed
       - If media surface: _sound_was_media_obstructed
    3. If listener and source in different media:
       - _sound_was_media_muffled

Obstructed sounds:
  - Volume reduced by ~50%
  - High frequencies filtered (muffled)
```

### Ambient Sound System

**Ambient Sound Types** (28 defined):
| ID | Name | Loop Type |
|----|------|-----------|
| 0 | `_ambient_snd_water` | Continuous |
| 1 | `_ambient_snd_sewage` | Continuous |
| 2 | `_ambient_snd_lava` | Continuous |
| 3 | `_ambient_snd_goo` | Continuous |
| 4 | `_ambient_snd_under_media` | Continuous |
| 5 | `_ambient_snd_wind` | Continuous |
| 6 | `_ambient_snd_waterfall` | Continuous |
| 7 | `_ambient_snd_siren` | Continuous |
| 8 | `_ambient_snd_fan` | Continuous |
| 9 | `_ambient_snd_spht_door` | Continuous |
| 10 | `_ambient_snd_spht_platform` | Continuous |
| ... | ... | ... |

**Random Sound Types** (5 defined):
| ID | Name | Trigger |
|----|------|---------|
| 0 | `_random_snd_water_drip` | Random interval |
| 1 | `_random_snd_surface_explosion` | Random interval |
| 2 | `_random_snd_underground_explosion` | Random interval |
| 3 | `_random_snd_owl` | Random interval |
| 4 | `_random_snd_creak` | Random interval |

**Ambient Sound Pipeline**:
```
Per-Polygon Ambient Sound:
    1. Check polygon.ambient_sound_image_index
    2. If player in polygon with ambient sound:
       - Start/continue ambient playback on ambient channel
       - Volume based on player position within polygon
    3. If player leaves polygon:
       - Fade out ambient over ~0.5 seconds

Random sounds triggered at intervals defined per-sound
```

### Sound Permutations

Marathon supports multiple variations of each sound to prevent repetition:

```c
struct sound_definition {
    short sound_code;           // Unique identifier
    short behavior_index;       // How sound behaves (looping, etc.)
    word flags;                 // Various flags
    word chance;                // Random playback chance [0, 65535]
    fixed low_pitch, high_pitch; // Pitch variation range
    short permutations;         // Number of variations
    short permutations_played;  // Bitmask of recently played
    // ... followed by permutation data
};

// Example: Rifle fire has 3 permutations
// Each play randomly selects one (avoiding recent repeats)
```

---

## 14. Items & Inventory System

### Item Categories

**Item Kinds** (`items.h:8-19`):
```c
enum { /* item types (class) */
    _weapon,           // 0 - Pickable weapons
    _ammunition,       // 1 - Ammo pickups
    _powerup,          // 2 - Special abilities (invisibility, etc.)
    _item,             // 3 - Key items (keys, chips)
    _weapon_powerup,   // 4 - Extra ammo from weapon pickup
    _ball,             // 5 - Multiplayer game balls

    NUMBER_OF_ITEM_TYPES,
    _network_statistics = NUMBER_OF_ITEM_TYPES  // Used in game_window.c
};
```

### Complete Item Types Table

**All 38 Item Types** (`items.h:21-64`):

| ID | Enum Constant | Name | Kind | Notes |
|----|---------------|------|------|-------|
| 0 | `_i_knife` | Fist | Weapon | Always available |
| 1 | `_i_magnum` | Magnum Pistol | Weapon | .44 Magnum |
| 2 | `_i_magnum_magazine` | Magnum Clip | Ammo | 8 rounds |
| 3 | `_i_plasma_pistol` | Fusion Pistol | Weapon | Energy weapon |
| 4 | `_i_plasma_magazine` | Fusion Battery | Ammo | 20 units |
| 5 | `_i_assault_rifle` | MA-75B | Weapon | With grenade launcher |
| 6 | `_i_assault_rifle_magazine` | MA-75B Clip | Ammo | 52 rounds |
| 7 | `_i_assault_grenade_magazine` | Grenades | Ammo | 7 grenades |
| 8 | `_i_missile_launcher` | SPNKR | Weapon | Rocket launcher |
| 9 | `_i_missile_launcher_magazine` | Rockets | Ammo | 2 rockets |
| 10 | `_i_invisibility_powerup` | Invisibility | Powerup | Temporary cloak |
| 11 | `_i_invincibility_powerup` | Invincibility | Powerup | Temporary god mode |
| 12 | `_i_infravision_powerup` | Infravision | Powerup | See in dark |
| 13 | `_i_alien_shotgun` | Alien Weapon | Weapon | Pfhor weapon |
| 14 | `_i_alien_shotgun_magazine` | Alien Ammo | Ammo | Variable |
| 15 | `_i_flamethrower` | TOZT-7 | Weapon | Flamethrower |
| 16 | `_i_flamethrower_canister` | Napalm | Ammo | 210 ticks fuel |
| 17 | `_i_extravision_powerup` | Extravision | Powerup | Wide FOV |
| 18 | `_i_oxygen_powerup` | Oxygen | Powerup | Refill O2 |
| 19 | `_i_energy_powerup` | 1x Health | Powerup | Restore 1 bar |
| 20 | `_i_double_energy_powerup` | 2x Health | Powerup | Restore 2 bars |
| 21 | `_i_triple_energy_powerup` | 3x Health | Powerup | Restore 3 bars |
| 22 | `_i_shotgun` | WSTE-M5 | Weapon | Dual-wield shotgun |
| 23 | `_i_shotgun_magazine` | Shotgun Shells | Ammo | 2 shells |
| 24 | `_i_spht_door_key` | S'pht Key | Item | Opens doors |
| 25 | `_i_uplink_chip` | Uplink Chip | Item | Mission item |
| 26+ | `_i_light_blue_ball` | Light Blue Ball | Ball | CTF/Ball games |
| 27 | `_i_red_ball` | Red Ball | Ball | Team games |
| 28 | `_i_violet_ball` | Violet Ball | Ball | Team games |
| 29 | `_i_yellow_ball` | Yellow Ball | Ball | Team games |
| 30 | `_i_brown_ball` | Brown Ball | Ball | Team games |
| 31 | `_i_orange_ball` | Orange Ball | Ball | Team games |
| 32 | `_i_blue_ball` | Blue Ball | Ball | Team games |
| 33 | `_i_green_ball` | Green Ball | Ball | Team games |
| 34 | `_i_smg` | KKV-7 SMG | Weapon | Submachine gun |
| 35 | `_i_smg_ammo` | SMG Ammo | Ammo | 32 rounds |

### Item Definition Structure

```c
struct item_definition {
    short item_kind;               // Category from above
    short singular_name_id;        // String resource ID
    short plural_name_id;          // String resource ID
    shape_descriptor base_shape;   // Visual appearance
    short maximum_count_per_player; // Inventory limit
    short invalid_environments;    // Where item cannot exist
};
```

**Environment Restrictions**:
```c
enum {
    _environment_normal = 0x0000,
    _environment_vacuum = 0x0001,        // No oxygen
    _environment_magnetic = 0x0002,      // Compass interference
    _environment_rebellion = 0x0004,     // Storyline flag
    _environment_low_gravity = 0x0008,   // Reduced gravity
    _environment_network = 0x2000,       // Multiplayer only
    _environment_single_player = 0x4000  // Single-player only
};
```

### Complete Item Table

| ID | Item | Kind | Max | Notes |
|----|------|------|-----|-------|
| 0 | Fist | weapon | 1 | Always available |
| 1 | Magnum Pistol | weapon | 2 | Dual-wield capable |
| 2 | Magnum Magazine | ammunition | 8 | |
| 3 | Plasma Pistol | weapon | 1 | |
| 4 | Plasma Energy Cell | ammunition | 8 | |
| 5 | Assault Rifle | weapon | 1 | |
| 6 | AR Magazine | ammunition | 8 | |
| 7 | AR Grenades | ammunition | 8 | Secondary ammo |
| 8 | Missile Launcher | weapon | 1 | |
| 9 | Missile 2-Pack | ammunition | 4 | |
| 10 | Invisibility | powerup | 1 | Timed effect |
| 11 | Invincibility | powerup | 1 | Timed effect |
| 12 | Infravision | powerup | 1 | Night vision |
| 13 | Alien Shotgun | weapon | 2 | Enemy weapon |
| 14 | Alien Shotgun Ammo | ammunition | 8 | |
| 15 | Flamethrower | weapon | 1 | |
| 16 | Flamethrower Canister | ammunition | 8 | |
| 17 | Extravision | powerup | 1 | Wide FOV |
| 18 | Oxygen | item | 1 | Refills O2 |
| 19 | Energy x1 | item | 1 | Restores shields |
| 20 | Energy x2 | item | 1 | Double shields |
| 21 | Energy x3 | item | 1 | Triple shields |
| 22 | Shotgun | weapon | 2 | Dual-wield capable |
| 23 | Shotgun Shells | ammunition | 8 | |
| 24 | S'pht Key | item | 1 | Key item |
| 25 | Uplink Chip | item | 1 | For terminals |
| 26 | Ball (Red) | ball | 1 | Multiplayer |
| 27-33 | Ball (Colors) | ball | 1 | Multiplayer |

### Item Pickup System

**Pickup Detection**:
```c
#define MAXIMUM_ARM_REACH (3*WORLD_ONE_FOURTH)  // ~768 units

// Player can pick up items within arm's reach
boolean try_and_get_item(short player_index, short polygon_index) {
    for (each object in polygon) {
        if (object is item && distance < MAXIMUM_ARM_REACH) {
            return get_item(player_index, object_index);
        }
    }
}
```

**Pickup Flow**:
```
Player approaches item:
    │
    ├─► Check distance < MAXIMUM_ARM_REACH
    │
    ├─► Validate environment compatibility
    │     └─► Network-only items blocked in single-player
    │     └─► Single-player items blocked in multiplayer
    │
    ├─► Check inventory space
    │     └─► maximum_count_per_player limit
    │     └─► Total Carnage: unlimited ammo
    │
    ├─► Add to inventory
    │     └─► Weapons: process_new_item_for_reloading()
    │     └─► Ammo: add to count
    │     └─► Powerups: start timer
    │
    ├─► Play pickup sound
    │
    └─► Remove item from world
        └─► May respawn later (see Object Placement)
```

### Powerup Timers

| Powerup | Duration | Effect |
|---------|----------|--------|
| Invisibility | ~30 seconds | Semi-transparent, AI ignores |
| Invincibility | ~30 seconds | No damage (except telefrag) |
| Infravision | ~60 seconds | See in darkness |
| Extravision | Permanent | 130° FOV |

---

## 15. Control Panels System

Control panels are interactive wall surfaces that provide various functions.

### Panel Classes

```c
enum {  // control_panel_class
    _panel_is_oxygen_refuel,       // Restores oxygen
    _panel_is_shield_refuel,       // Restores shields (1x)
    _panel_is_double_shield_refuel, // Restores shields (2x speed)
    _panel_is_triple_shield_refuel, // Restores shields (3x speed)
    _panel_is_light_switch,        // Toggles light source
    _panel_is_platform_switch,     // Activates platform/door
    _panel_is_tag_switch,          // Triggers tagged objects
    _panel_is_pattern_buffer,      // Save game point
    _panel_is_computer_terminal    // Information terminal
};
```

### Panel Definition Structure

```c
struct control_panel_definition {
    short panel_class;              // Type from above
    word flags;                     // Behavior flags

    short collection;               // Texture collection
    short active_shape;             // Texture when active/on
    short inactive_shape;           // Texture when inactive/off

    short sounds[3];                // Activating, Deactivating, Unusable
    fixed sound_frequency;          // Pitch modifier

    short item;                     // Required item (NONE = no requirement)
};
```

### Activation System

**Activation Constants**:
```c
#define MAXIMUM_ACTIVATION_RANGE (3*WORLD_ONE)        // General activation
#define MAXIMUM_PLATFORM_ACTIVATION_RANGE (3*WORLD_ONE)
#define MAXIMUM_CONTROL_ACTIVATION_RANGE (WORLD_ONE+WORLD_ONE_HALF)  // ~1536 units
#define MINIMUM_RESAVE_TICKS (2*TICKS_PER_SECOND)     // Pattern buffer cooldown
```

**Activation Flow**:
```
Player presses Action key:
    │
    ├─► find_action_key_target()
    │     ├─► Search lines within MAXIMUM_CONTROL_ACTIVATION_RANGE
    │     ├─► Check line has control panel side
    │     └─► Verify player facing panel
    │
    ├─► line_side_has_control_panel()
    │     └─► Return side_index_with_panel
    │
    └─► change_panel_state()
          │
          ├─► _panel_is_oxygen_refuel:
          │     └─► Add oxygen each tick while holding action
          │
          ├─► _panel_is_shield_refuel:
          │     └─► Add shields each tick (rate varies by type)
          │
          ├─► _panel_is_light_switch:
          │     └─► Toggle light on/off
          │
          ├─► _panel_is_platform_switch:
          │     └─► Activate/deactivate platform
          │
          ├─► _panel_is_tag_switch:
          │     ├─► Check required item (uplink chip)
          │     └─► Trigger all objects with matching tag
          │
          ├─► _panel_is_pattern_buffer:
          │     ├─► Check MINIMUM_RESAVE_TICKS cooldown
          │     └─► Save game state
          │
          └─► _panel_is_computer_terminal:
                └─► Enter terminal interface mode
```

### Recharge Stations

**Recharge Rates**:
| Panel Type | Rate | Notes |
|------------|------|-------|
| Oxygen Refuel | Full/second | Restores O2 |
| Shield (1x) | FIXED_ONE/tick | Normal speed |
| Shield (2x) | FIXED_ONE+FIXED_ONE/8 | 12.5% faster |
| Shield (3x) | FIXED_ONE+FIXED_ONE/4 | 25% faster |

**Recharge Visualization**:
```
Oxygen Station:             Shield Station:
┌──────────────┐           ┌──────────────┐
│ ░░░░░░░░░░░░ │           │ ▓▓▓▓▓▓▓▓▓▓▓▓ │
│ ░░ O2  ░░░░ │           │ ▓▓ ⚡ ▓▓▓▓▓ │
│ ░░ REFUEL░░ │           │ ▓▓ CHARGE ▓▓ │
│ ░░░░░░░░░░░░ │           │ ▓▓▓▓▓▓▓▓▓▓▓▓ │
└──────────────┘           └──────────────┘

Player holds Action:        Player holds Action:
  O2: ████████░░ → ██████████   Shields: ███░░░░░░░ → █████████░
```

### Switch States

Switches maintain state and can be toggled:

```c
boolean switch_can_be_toggled(short line_index, boolean player_hit) {
    // Check if switch is not permanently locked
    // Check if linked object can accept toggle
    // For platform switches: check platform state
    // For light switches: check light state
}

void set_control_panel_texture(struct side_data *side) {
    // Update texture based on current state
    // active_shape if ON, inactive_shape if OFF
}
```

---

## 16. Damage System

### Damage Definition Structure

```c
struct damage_definition {
    short type;    // Damage type (see types below)
    short flags;   // Modifier flags

    short base;    // Base damage amount
    short random;  // Random additional damage [0, random)
    fixed scale;   // Multiplier (FIXED_ONE = 1.0)
};
```

### Damage Calculation Formula

```c
short calculate_damage(struct damage_definition *damage) {
    // Step 1: Base + random
    short total_damage = damage->base + (damage->random ? random() % damage->random : 0);

    // Step 2: Apply scale
    total_damage = FIXED_INTEGERAL_PART(total_damage * damage->scale);

    // Step 3: Difficulty modifier (alien damage only)
    if (damage->flags & _alien_damage) {
        switch (difficulty_level) {
            case _wuss_level: total_damage -= total_damage >> 1; break;  // -50%
            case _easy_level: total_damage -= total_damage >> 2; break;  // -25%
            // Normal and above: no reduction
        }
    }

    return total_damage;
}
```

**Damage Formula Diagram**:
```
Damage Calculation Pipeline:

  base (100) ─────┐
                  │
  random (50) ────┼──► Roll: 100 + rand(0,49) = 125
                  │
                  ▼
            ┌─────────────┐
            │ Apply Scale │  scale = 1.5 (FIXED_ONE*3/2)
            └─────────────┘
                  │
                  ▼
            125 × 1.5 = 187
                  │
                  ▼
            ┌─────────────┐
            │  Difficulty │  (if alien damage)
            │  Modifier   │
            └─────────────┘
                  │
       ┌──────────┼──────────┐
       │          │          │
    Wuss       Normal     Major
    -50%        0%         0%
       │          │          │
      93        187        187
       │          │          │
       └──────────┴──────────┘
                  │
                  ▼
            Final Damage
```

### Monster Damage Application

```c
void damage_monster(short target_index, short aggressor_index,
                    struct damage_definition *damage) {
    short delta_vitality = calculate_damage(damage);

    // Immunity check
    if (definition->immunities & FLAG(damage->type)) {
        return;  // No damage
    }

    // Weakness check (2x damage)
    if (definition->weaknesses & FLAG(damage->type)) {
        delta_vitality <<= 1;
    }

    // Apply damage
    monster->vitality -= delta_vitality;

    // Check for death
    if (monster->vitality <= 0) {
        kill_monster(target_index, aggressor_index, damage);
    }
}
```

### Player Damage Response

```c
struct damage_response_definition {
    short type;          // Damage type this responds to
    short damage_threshhold;  // Minimum damage for response
    word fade;           // Screen fade color
    short sound;         // Pain sound to play
    fixed death_sound_chance;  // Probability of death sound
    short death_action;  // What happens on death
};
```

**Player Shield System**:
```
Damage to Player:
    │
    ├─► Check Invincibility
    │     └─► If active AND not fusion bolt: absorb all
    │
    ├─► Apply to Shields first
    │     └─► shields -= damage
    │
    ├─► If shields < 0:
    │     └─► Overflow goes to health
    │     └─► dead_player() if health <= 0
    │
    └─► Trigger damage response
          ├─► Screen fade (red flash)
          ├─► Pain sound
          └─► Knockback
```

### Damage Types Reference

| Type ID | Name | Typical Source |
|---------|------|----------------|
| 0 | `_damage_explosion` | Rockets, grenades |
| 1 | `_damage_electrical_staff` | Staff weapon |
| 2 | `_damage_projectile` | Bullets |
| 3 | `_damage_absorbed` | Shield blocked |
| 4 | `_damage_flame` | Flamethrower |
| 5 | `_damage_hound_claws` | Hound melee |
| 6 | `_damage_alien_projectile` | Alien weapons |
| 7 | `_damage_hulk_slap` | Hulk melee |
| 8 | `_damage_compiler_bolt` | Compiler attack |
| 9 | `_damage_fusion_bolt` | Fusion pistol |
| 10 | `_damage_hunter_bolt` | Hunter shot |
| 11 | `_damage_fist` | Player punch |
| 12 | `_damage_teleporter` | Telefrag |
| 13 | `_damage_defender` | Defender attack |
| 14 | `_damage_yeti_claws` | Yeti melee |
| 15 | `_damage_yeti_projectile` | Yeti shot |
| 16 | `_damage_crushing` | Platform/door |
| 17 | `_damage_lava` | Lava contact |
| 18 | `_damage_suffocation` | No oxygen |
| 19 | `_damage_goo` | Sewage/goo |
| 20 | `_damage_energy_drain` | Shield drain |
| 21 | `_damage_oxygen_drain` | O2 drain |
| 22 | `_damage_hummer_bolt` | Hummer attack |
| 23 | `_damage_shotgun_projectile` | Shotgun pellets |

---

## 17. Multiplayer Game Types

### Game Type Enumeration

```c
enum {
    _game_of_kill_monsters,      // Co-op: Kill aliens
    _game_of_cooperative_play,   // Co-op: Story mode
    _game_of_capture_the_flag,   // CTF
    _game_of_king_of_the_hill,   // KOTH
    _game_of_kill_man_with_ball, // Ball carrier scores
    _game_of_tag,                // Avoid being "it"
    _game_of_defense,            // Attack/Defend
    _game_of_rugby               // Score goals
};
```

### Game Type Details

#### Kill Monsters (Every Man for Himself)
```
Objective: Highest kill count wins
Scoring: kills - deaths
```

#### Cooperative Play
```
Objective: Complete level together
Scoring: Percentage of total monster damage dealt
         ranking = (100 * player_monster_damage) / total_monster_damage
```

#### Capture the Flag
```
Objective: Capture enemy flags
Scoring: Number of flag pulls
         ranking = player->netgame_parameters[_flag_pulls]

Mechanics:
  - Each team has base polygon (_polygon_is_base)
  - Ball items represent flags
  - Carry enemy flag to your base
```

#### King of the Hill
```
Objective: Occupy hill polygon longest
Scoring: Ticks spent in hill
         ranking = player->netgame_parameters[_king_of_hill_time]

Hill Location:
  - Polygons marked _polygon_is_hill
  - Compass points to hill center
  - Multiple polygons = centroid

Beacon Calculation:
  x = sum(polygon.center.x) / count
  y = sum(polygon.center.y) / count
```

#### Kill Man With Ball
```
Objective: Kill the ball carrier
Scoring: Ticks holding ball
         ranking = player->netgame_parameters[_ball_carrier_time]

Mechanics:
  - Single ball spawns
  - Carrier is visible to all (compass)
  - Carrier earns points over time
  - Killing carrier drops ball
```

#### Tag
```
Objective: Avoid being "it"
Scoring: NEGATIVE time spent "it"
         ranking = -player->netgame_parameters[_time_spent_it]

Mechanics:
  - One player is "it"
  - Tag another to transfer
  - "It" is visible on compass
  - Lowest time wins
```

#### Defense (Offense/Defense)
```
Objective: Attack or defend based on team
Scoring: kills - deaths + 50 (if winning team)

Parameters:
  - _defending_team: Which team defends
  - _maximum_offender_time_in_base: Win condition
```

#### Rugby
```
Objective: Score goals with ball
Scoring: Goals scored + (kills - deaths)
         ranking = player->netgame_parameters[_points_scored] + kills - deaths
```

### Network Compass System

```c
short get_network_compass_state(short player_index) {
    // Returns bitmask of compass quadrants
    // _network_compass_ne, _network_compass_nw
    // _network_compass_se, _network_compass_sw
    // _network_compass_all_on, _network_compass_all_off
}
```

**Compass Visualization**:
```
Compass Display:
    ┌───────┐
    │ NW NE │
    │   ●   │   ● = objective direction
    │ SW SE │
    └───────┘

Example: Objective is northeast
    ┌───────┐
    │ ░░ ▓▓ │   NE quadrant highlighted
    │   ●   │
    │ ░░ ░░ │
    └───────┘
```

---

## 18. Random Number Generation

Marathon uses a Linear Feedback Shift Register (LFSR) for deterministic random numbers.

### Dual RNG System

**Two separate generators**:
```c
static word random_seed = 0x1;       // Network-synchronized
static word local_random_seed = 0x1; // Local-only (visual effects)
```

### LFSR Implementation

```c
word random(void) {
    word seed = random_seed;

    if (seed & 1) {
        seed = (seed >> 1) ^ 0xb400;  // Feedback polynomial
    } else {
        seed >>= 1;
    }

    return (random_seed = seed);
}

word local_random(void) {
    word seed = local_random_seed;

    if (seed & 1) {
        seed = (seed >> 1) ^ 0xb400;
    } else {
        seed >>= 1;
    }

    return (local_random_seed = seed);
}
```

**LFSR Visualization**:
```
16-bit LFSR with polynomial 0xb400:

Bit positions:  15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
                ─────────────────────────────────────────────────
Initial:         0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  1

If bit 0 = 1:
  1. Shift right:  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0  0
  2. XOR 0xb400:   1  0  1  1  0  1  0  0  0  0  0  0  0  0  0  0
     Result:       1  0  1  1  0  1  0  0  0  0  0  0  0  0  0  0

If bit 0 = 0:
  1. Shift right only

0xb400 = 1011 0100 0000 0000
         ↑  ↑  ↑
        15 13 12 10 (feedback taps)

Period: 65535 (2^16 - 1, maximum for 16-bit LFSR)
```

### Usage Guidelines

| Function | Use Case | Network Safe? |
|----------|----------|---------------|
| `random()` | Gameplay mechanics | YES |
| `local_random()` | Visual effects | NO |

**Examples**:
```c
// CORRECT: Damage rolls use synchronized random
damage = base + random() % random_damage;

// CORRECT: Particle effects use local random
particle_x_offset = local_random() % spread;

// WRONG: Using local_random for gameplay
// This would cause network desync!
```

### Seed Synchronization

```c
void set_random_seed(word seed) {
    random_seed = seed ? seed : 1;  // Never allow 0 (LFSR would stick)
}

word get_random_seed(void) {
    return random_seed;
}
```

**Network Sync**:
```
Game Start:
    Host generates seed ──────────► All clients set same seed

During Game:
    All clients call random() in identical order
    Same sequence on all machines
    Deterministic simulation maintained
```

---

## 19. Shape Animation System

> **🔧 For Porting:** Replace `shapes_macintosh.c` with code that reads shape collections from the binary Shapes file using `fopen()`/`fread()`. The data structures in `shapes.c` and `shapes.h` are portable. Convert 8-bit palette colors to 32-bit ARGB during load time for modern rendering.

### Collection Index

**All 32 Shape Collections** (`shape_descriptors.h:29-60`):

| ID | Enum Constant | Contents | Type |
|----|---------------|----------|------|
| 0 | `_collection_interface` | HUD, menus | Interface |
| 1 | `_collection_weapons_in_hand` | First-person weapons | Object |
| 2 | `_collection_juggernaut` | Juggernaut sprites | Object |
| 3 | `_collection_tick` | Tick sprites | Object |
| 4 | `_collection_rocket` | Rockets, explosions, effects | Object |
| 5 | `_collection_hunter` | Hunter sprites | Object |
| 6 | `_collection_player` | Player body sprites | Object |
| 7 | `_collection_items` | Pickups, weapons on ground | Object |
| 8 | `_collection_trooper` | Trooper sprites | Object |
| 9 | `_collection_fighter` | Fighter sprites | Object |
| 10 | `_collection_defender` | Defender sprites | Object |
| 11 | `_collection_yeti` | Yeti sprites (all variants) | Object |
| 12 | `_collection_civilian` | Civilian sprites | Object |
| 13 | `_collection_vacuum_civilian` | Vacuum civilian | Object |
| 14 | `_collection_enforcer` | Enforcer sprites | Object |
| 15 | `_collection_hummer` | Hummer sprites | Object |
| 16 | `_collection_compiler` | Compiler sprites | Object |
| 17 | `_collection_walls1` | Water environment textures | Wall |
| 18 | `_collection_walls2` | Lava environment textures | Wall |
| 19 | `_collection_walls3` | Sewage environment textures | Wall |
| 20 | `_collection_walls4` | Jjaro environment textures | Wall |
| 21 | `_collection_walls5` | Pfhor environment textures | Wall |
| 22 | `_collection_scenery1` | Water scenery | Scenery |
| 23 | `_collection_scenery2` | Lava scenery | Scenery |
| 24 | `_collection_scenery3` | Sewage scenery | Scenery |
| 25 | `_collection_scenery4` | Jjaro (Pathways) scenery | Scenery |
| 26 | `_collection_scenery5` | Pfhor (Alien) scenery | Scenery |
| 27 | `_collection_landscape1` | Day landscape | Landscape |
| 28 | `_collection_landscape2` | Night landscape | Landscape |
| 29 | `_collection_landscape3` | Moon landscape | Landscape |
| 30 | `_collection_landscape4` | Space landscape | Landscape |
| 31 | `_collection_cyborg` | Cyborg sprites | Object |

### Collection Structure

**Collection Types**:
```c
enum {
    _unused_collection = 0,     // Raw data
    _wall_collection,           // Wall textures (raw)
    _object_collection,         // Sprites (RLE compressed)
    _interface_collection,      // UI elements (raw)
    _scenery_collection         // Scenery sprites (RLE)
};
```

### Animation Hierarchy

```
Collection (e.g., "Hunter" collection)
    │
    ├─► High-Level Shapes (animations)
    │     │
    │     ├─► "Walking" (8 views × 12 frames)
    │     ├─► "Attacking" (8 views × 6 frames)
    │     ├─► "Dying" (1 view × 8 frames)
    │     └─► ...
    │
    ├─► Low-Level Shapes (individual frames)
    │     │
    │     ├─► Frame 0: bitmap 5, origin (32, 64)
    │     ├─► Frame 1: bitmap 5, origin (33, 65), mirrored
    │     └─► ...
    │
    └─► Bitmaps (pixel data)
          │
          ├─► Bitmap 0: 64×128, RLE compressed
          ├─► Bitmap 1: 64×128, RLE compressed
          └─► ...
```

### High-Level Shape Definition

```c
struct high_level_shape_definition {
    short type;                     // Always 0
    word flags;                     // Animation flags

    char name[33];                  // "Walking", "Attacking", etc.

    short number_of_views;          // 1, 3, 5, or 8 viewing angles
    short frames_per_view;          // Frames in animation
    short ticks_per_frame;          // Animation speed
    short key_frame;                // Important frame (attack moment)

    short transfer_mode;            // Special rendering mode
    short transfer_mode_period;     // Effect cycle time

    short first_frame_sound;        // Sound at animation start
    short key_frame_sound;          // Sound at key frame
    short last_frame_sound;         // Sound at animation end

    short pixels_to_world;          // Scale factor
    short loop_frame;               // Where to loop back

    // Followed by: number_of_views × frames_per_view indices
    short low_level_shape_indexes[];
};
```

**View Angles**:
```
8-View System:
                 View 2
                   │
           View 3  │  View 1
                 ╲ │ ╱
                  ╲│╱
        View 4 ────●──── View 0 (facing camera)
                  ╱│╲
                 ╱ │ ╲
           View 5  │  View 7
                   │
                 View 6

Index calculation:
  view_index = ((facing - viewer_facing + QUARTER_CIRCLE) × number_of_views + HALF_CIRCLE) / FULL_CIRCLE
```

### Low-Level Shape Definition

```c
struct low_level_shape_definition {
    word flags;                     // Mirroring and obscured flags
    fixed minimum_light_intensity;  // Self-illumination minimum

    short bitmap_index;             // Which bitmap to use

    short origin_x, origin_y;       // Sprite anchor point
    short key_x, key_y;             // Registration point

    short world_left, world_right;  // World-space bounds
    short world_top, world_bottom;
    short world_x0, world_y0;       // World offset
};
```

**Flags**:
```c
#define _X_MIRRORED_BIT 0x8000        // Horizontally flip
#define _Y_MIRRORED_BIT 0x4000        // Vertically flip
#define _KEYPOINT_OBSCURED_BIT 0x2000 // Key point hidden
```

**Mirroring Optimization**:
```
Original sprite:       X-Mirrored:
┌─────────────┐       ┌─────────────┐
│ ►           │       │           ◄ │
│ ╔═══        │   →   │        ═══╗ │
│ ║           │       │           ║ │
│ ╠═══        │       │        ═══╣ │
│ ║           │       │           ║ │
└─────────────┘       └─────────────┘

Same bitmap, different render - saves memory!
```

### Animation Playback

```c
// Animation update (per tick)
void animate_object(struct object_data *object) {
    struct high_level_shape_definition *animation = get_animation(object);

    object->animation_tick++;

    if (object->animation_tick >= animation->ticks_per_frame) {
        object->animation_tick = 0;
        object->current_frame++;

        // Key frame actions
        if (object->current_frame == animation->key_frame) {
            play_sound(animation->key_frame_sound);
            trigger_key_frame_event(object);  // Attack damage, etc.
        }

        // Loop or end
        if (object->current_frame >= animation->frames_per_view) {
            if (animation->loop_frame != NONE) {
                object->current_frame = animation->loop_frame;
            } else {
                finish_animation(object);
            }
        }
    }
}
```

**Animation Timeline**:
```
Walking Animation (8 frames, 2 ticks/frame):

Frame:    0   1   2   3   4   5   6   7   0   1   ...
Tick:     0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 ...
          └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─────────
           Frame duration: 2 ticks each    Loop

Attack Animation (6 frames, key_frame = 3):

Frame:    0   1   2   3   4   5   (end)
                      ↑
                  Key frame
                  (damage applied)
                  (sound played)
```

### Shading Table System

```c
// Build shading tables for a collection
void build_shading_tables(struct rgb_color_value *colors, short color_count) {
    // Create gradient tables from full bright to full dark
    // number_of_shading_tables = 32 (8-bit) or 64 (16/32-bit)

    for (table = 0; table < number_of_shading_tables; table++) {
        fixed intensity = (table * FIXED_ONE) / number_of_shading_tables;

        for (color = 0; color < color_count; color++) {
            shading_table[table][color] = scale_color(colors[color], intensity);
        }
    }
}
```

**Shading Table Visualization**:
```
                Brightness Level
                0        16        31
Color Index   (Dark)   (Mid)   (Bright)
    0         [  0  ] [  0  ] [  0  ]    (Black stays black)
    1         [ 12  ] [ 64  ] [ 128 ]
    2         [ 20  ] [ 80  ] [ 160 ]
    ...
    255       [ 127 ] [ 192 ] [ 255 ]    (White scales)

Usage: shaded_pixel = shading_table[light_level][original_pixel]
```

---

## 20. Automap/Overhead Map System

The automap provides a top-down view of explored areas, helping players navigate Marathon's complex 3D environments.

### Exploration Tracking

Marathon tracks which parts of the level the player has explored using compact bitmask arrays.

**Automap Storage** (from `map.h`):
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

**Memory Layout**:
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

**Exploration Triggers**:
- When player enters a polygon: `ADD_POLYGON_TO_AUTOMAP(polygon_index)`
- When player can see a line: `ADD_LINE_TO_AUTOMAP(line_index)`
- Rendering marks endpoints with flag `_endpoint_on_automap = 0x2000`

### Overhead Map Data Structure

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

### Rendering Modes

```c
enum /* overhead map modes */ {
    _rendering_saved_game_preview,  // Thumbnail in save dialog
    _rendering_checkpoint_map,      // Terminal checkpoint display
    _rendering_game_map             // Live gameplay map
};
```

**Mode Differences**:
| Mode | Shows Explored Only | Shows Player | Shows Entities | Interactive |
|------|---------------------|--------------|----------------|-------------|
| `_rendering_game_map` | Yes | Yes | Yes | Yes |
| `_rendering_checkpoint_map` | Uses false automap | No | Checkpoint only | No |
| `_rendering_saved_game_preview` | Yes | Yes | No | No |

### Scale System

```c
#define OVERHEAD_MAP_MINIMUM_SCALE 1   // Zoomed out (overview)
#define OVERHEAD_MAP_MAXIMUM_SCALE 4   // Zoomed in (detail)
#define DEFAULT_OVERHEAD_MAP_SCALE 3

#define WORLD_TO_SCREEN_SCALE_ONE 8

// Convert world coordinates to screen coordinates
#define WORLD_TO_SCREEN(x, x0, scale) \
    (((x)-(x0))>>(WORLD_TO_SCREEN_SCALE_ONE-(scale)))
```

**Scale Visualization**:
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

### Color System

**Polygon Colors** (area fills):
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

**Line Colors** (edges):
```c
enum /* line colors */ {
    _solid_line_color,         // Impassable walls - bright
    _elevation_line_color,     // Height change - medium
    _control_panel_line_color  // Interactive panels - highlighted
};
```

**Entity Colors** (things):
```c
enum /* thing colors */ {
    _civilian_thing,    // Friendly BOBs - green
    _item_thing,        // Pickups - yellow
    _monster_thing,     // Enemies - red
    _projectile_thing,  // Bullets/rockets - white
    _checkpoint_thing   // Terminal checkpoint - special
};
```

### Rendering Pipeline

```
overhead_map_begin():
    │
    ├─► Save graphics state
    ├─► Set clipping rectangle
    └─► Clear map area to background color

overhead_map_end():
    │
    ├─► Restore graphics state
    └─► Draw any overlays

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
```

### Coordinate Transformation

**World to Screen Conversion**:
```c
// Transform a world point to map screen coordinates
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

**Example**:
```
Player at world (4096, 6144), origin at (4096, 4096), scale 3:

X offset: (4096 - 4096) >> (8-3) = 0 >> 5 = 0
Y offset: (6144 - 4096) >> (8-3) = 2048 >> 5 = 64

Screen position: (left + half_width + 0, top + half_height - 64)
                = (100 + 64 + 0, 100 + 64 - 64)
                = (164, 100)

Player appears 64 pixels above center of map
```

### Entity Display

**Player Marker**:
```
Arrow showing player position and facing:

        ▲
       ╱│╲
      ╱ │ ╲
     ╱  │  ╲     Arrow points in facing direction
    ╱   │   ╲    Size scales with zoom level
   ╱    │    ╲
  ╱     │     ╲
 ────────────────

Arrow rotates based on player->facing angle
```

**Monster/Item Markers**:
```
Small dots or shapes indicating entity type:

    ●  Monster (red)
    ◆  Item (yellow)
    ○  Projectile (white)
    ★  Checkpoint (highlighted)
```

### False Automap (Checkpoint Maps)

When rendering checkpoint maps from terminals, Marathon creates a temporary "false automap":

```c
// Terminal shows specific area without revealing actual exploration
void draw_checkpoint_map(short checkpoint_polygon) {
    // Save real automap state
    byte *saved_lines = automap_lines;
    byte *saved_polygons = automap_polygons;

    // Create false automap showing checkpoint area
    automap_lines = allocate_temporary_lines();
    automap_polygons = allocate_temporary_polygons();

    // Flood fill from checkpoint to mark visible area
    mark_checkpoint_visible_area(checkpoint_polygon);

    // Render with false automap
    draw_overhead_map();

    // Restore real automap
    automap_lines = saved_lines;
    automap_polygons = saved_polygons;
}
```

### Persistence

Automap state is saved with the game:

```c
// Save game tags for automap
AUTOMAP_LINES     // Stores automap_lines array
AUTOMAP_POLYGONS  // Stores automap_polygons array

// Loading restores exploration progress
// New game initializes arrays to zero (nothing explored)
```

---

## 21. HUD Rendering System

> **🔧 For Porting:** HUD rendering in `interface.c` uses QuickDraw calls (`PaintRect`, `DrawText`, etc.). Replace with direct framebuffer writes or your UI library. The motion sensor (`motion_sensor.c`) is pure math—just needs pixel plotting. Font rendering needs a replacement (use stb_truetype, bitmap fonts, or microui).

The Heads-Up Display (HUD) provides vital information overlaid on the game view.

### HUD Layout

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│                     GAME VIEW                               │
│                                                             │
│                                                             │
│                                                             │
│                                                             │
│                                                             │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ ┌─────────┐ ┌─────────────────────────────────┐ ┌─────────┐│
│ │ WEAPON  │ │       MOTION SENSOR             │ │  AMMO   ││
│ │  PANEL  │ │        (Radar)                  │ │  PANEL  ││
│ │         │ │                                 │ │         ││
│ ├─────────┤ │            ●                    │ ├─────────┤│
│ │ SHIELDS │ │           /●\                   │ │  OXYGEN ││
│ │ ████████│ │          ●   ●                  │ │ ████████││
│ └─────────┘ └─────────────────────────────────┘ └─────────┘│
└─────────────────────────────────────────────────────────────┘
```

### HUD Elements

**1. Weapon Display**:
- Current weapon sprite (idle, firing, reloading)
- Weapon bob synchronized with movement
- Dual-wield positioning for akimbo weapons

**2. Shield/Health Bar**:
```c
// Shield display levels
#define MAXIMUM_PLAYER_SHIELDS (3*FIXED_ONE)  // 3x full charge possible

// Bar calculation
bar_width = (current_shields * MAX_BAR_WIDTH) / MAXIMUM_PLAYER_SHIELDS;
bar_color = get_shield_color(current_shields);  // Red/yellow/green gradient
```

**3. Oxygen Bar** (in vacuum/underwater):
```c
// Oxygen display
#define MAXIMUM_PLAYER_OXYGEN (5*TICKS_PER_SECOND*30)  // 5 minutes

// Only shown when relevant
if (environment_is_vacuum || player_head_below_media) {
    draw_oxygen_bar(player->oxygen);
}
```

**4. Ammunition Display**:
- Primary ammo count (magazine + reserve)
- Secondary ammo if applicable (grenades)
- Visual ammo counter or numeric display

**5. Motion Sensor** (covered in Section 8):
- Circular radar display
- Entity blips with trails
- Range: 8 WORLD_ONE radius

### Weapon Rendering

**Weapon Sprite Positioning**:
```c
struct weapon_display_information {
    short collection;          // Sprite collection
    short low_level_shape;     // Current frame
    fixed vertical_position;   // Screen Y offset
    fixed horizontal_position; // Screen X offset (dual-wield)
    short vertical_flip;       // Mirroring
    short horizontal_flip;
    fixed brightness;          // Lighting level
};
```

**Weapon States Affecting Display**:
```
_weapon_idle      → Idle sprite + bob
_weapon_firing    → Firing animation + recoil
_weapon_charging  → Charge-up effect (fusion)
_weapon_reloading → Reload animation sequence
_weapon_raising   → Weapon enters screen from below
_weapon_lowering  → Weapon exits screen downward
```

**Dual-Wield Positioning**:
```
Single weapon:           Dual weapons:
    Center                Left    Right
       │                   │        │
       ▼                   ▼        ▼
   ┌───────┐          ┌───────┐┌───────┐
   │Weapon │          │ Left  ││ Right │
   │Sprite │          │Weapon ││Weapon │
   └───────┘          └───────┘└───────┘
```

### Interface Panel Rendering

**Panel Structure**:
```c
// HUD panels are drawn from interface graphics collection
// Each panel element has:
- Background texture (metal panel frame)
- Dynamic content area (bars, numbers)
- Icon overlays (weapon icons, status indicators)
```

**Bar Rendering**:
```c
void draw_status_bar(
    short x, short y,           // Position
    short width, short height,  // Dimensions
    fixed current, fixed max,   // Value
    short full_color,           // Color when full
    short empty_color)          // Color when empty
{
    short fill_width = (current * width) / max;

    // Draw filled portion
    fill_rectangle(x, y, fill_width, height, full_color);

    // Draw empty portion
    fill_rectangle(x + fill_width, y, width - fill_width, height, empty_color);
}
```

### Network Game HUD Extensions

**Team Indicators**:
- Team color overlays
- Teammate health bars (if enabled)
- Score display

**Kill Feed** (multiplayer):
```
[Player1] killed [Player2]
[Player3] picked up Flag
```

### Font System

The font system provides text rendering for all 2D interface elements. It does NOT render 3D in-world text - all text in Marathon is on HUD, terminals, and menus.

**Source Files**: `screen_drawing.c`, `screen_drawing.h`

#### How the Font System Works

**Initialization** (`initialize_screen_drawing()`, screen_drawing.c:54):
```c
void initialize_screen_drawing(void) {
    // 1. Load interface rectangles from 'nrct' resource 128
    load_interface_rectangles();

    // 2. Load color palette from 'clut' resource 130
    load_screen_interface_colors();

    // 3. Load all 7 font specs from 'finf' resource 128
    for (loop = 0; loop < NUMBER_OF_INTERFACE_FONTS; ++loop) {
        GetNewTextSpec(&interface_fonts.fonts[loop], finfFONTS, loop);
        interface_fonts.heights[loop] = _get_font_height(&interface_fonts.fonts[loop]);
        interface_fonts.line_spacing[loop] = _get_font_line_spacing(&interface_fonts.fonts[loop]);
    }
}
```

**Font Storage Structure** (screen_drawing.c:29-34):
```c
struct interface_font_info {
    TextSpec fonts[NUMBER_OF_INTERFACE_FONTS];      // Mac font specs (ID, size, style)
    short heights[NUMBER_OF_INTERFACE_FONTS];       // Cached ascent + leading
    short line_spacing[NUMBER_OF_INTERFACE_FONTS];  // Cached ascent + descent + leading
};

static struct interface_font_info interface_fonts;  // Global font cache
```

#### Interface Fonts

```c
// From screen_drawing.h:67-76
enum { /* Fonts for the interface */
    _interface_font,              // 0 - General UI text
    _weapon_name_font,            // 1 - Weapon names in HUD
    _player_name_font,            // 2 - Player names (multiplayer)
    _interface_item_count_font,   // 3 - Item/ammo counts
    _computer_interface_font,     // 4 - Terminal body text
    _computer_interface_title_font, // 5 - Terminal titles
    _net_stats_font,              // 6 - Network game statistics
    NUMBER_OF_INTERFACE_FONTS     // 7
};
```

| ID | Font Constant | Used For | Typical Style |
|----|---------------|----------|---------------|
| 0 | `_interface_font` | General UI, menus | Medium sans-serif |
| 1 | `_weapon_name_font` | Weapon display | Bold, larger |
| 2 | `_player_name_font` | Multiplayer names | Medium |
| 3 | `_interface_item_count_font` | Ammo/item numbers | Numeric, fixed-width |
| 4 | `_computer_interface_font` | Terminal text | Monospace/typewriter |
| 5 | `_computer_interface_title_font` | Terminal headers | Bold monospace |
| 6 | `_net_stats_font` | Network stats | Small, readable |

#### Text Drawing Implementation

**Main Text Renderer** (`_draw_screen_text()`, screen_drawing.c:316-463):

```c
void _draw_screen_text(char *text, screen_rectangle *destination,
                       short flags, short font_id, short text_color) {
    // 1. Set font from cached specs
    SetFont(&interface_fonts.fonts[font_id]);

    // 2. Set text color from color table
    _get_interface_color(text_color, &new_color);
    RGBForeColor(&new_color);

    // 3. Handle word wrapping (if _wrap_text flag)
    if (flags & _wrap_text) {
        // Measure character-by-character
        while (count < strlen(text) && text_width < RECTANGLE_WIDTH(destination)) {
            text_width += CharWidth(text[count]);
            if (text[count] == ' ') last_space = count;  // Track last word break
            count++;
        }
        if (count != strlen(text)) {
            // Recursive call for remaining text on next line
            new_destination.top += line_spacing[font_id];
            _draw_screen_text(remaining_text, &new_destination, flags, font_id, text_color);
            text_to_draw[last_space] = 0;  // Truncate at word break
        }
    }

    // 4. Handle horizontal positioning
    text_width = TextWidth(text_to_draw, 0, strlen(text_to_draw));
    if (flags & _center_horizontal) {
        x = destination->left + (rect_width - text_width) / 2;
    } else if (flags & _right_justified) {
        x = destination->right - text_width;
    } else {
        x = destination->left;
    }

    // 5. Truncate if too wide
    if (text_width > RECTANGLE_WIDTH(destination)) {
        TruncText(RECTANGLE_WIDTH(destination), text_to_draw, &length, truncEnd);
    }

    // 6. Handle vertical positioning
    if (flags & _center_vertical) {
        y = destination->bottom - (rect_height - text_height) / 2;
    } else if (flags & _top_justified) {
        y = destination->top + text_height;
    } else {
        y = destination->bottom;  // Default: baseline at bottom
    }

    // 7. Draw the text
    MoveTo(x, y);
    DrawText(text_to_draw, 0, strlen(text_to_draw));
}
```

#### Text Measurement Functions

```c
// Get pixel width of string in specified font
short _text_width(char *buffer, short font_id) {
    SetFont(&interface_fonts.fonts[font_id]);
    return TextWidth(buffer, 0, strlen(buffer));  // Mac QuickDraw call
}

// Get line height for multi-line layout
short _get_font_line_height(short font_index) {
    return interface_fonts.line_spacing[font_index];  // Pre-cached value
}

// Internal: Calculate font height (ascent + leading)
static short _get_font_height(TextSpec *font) {
    FontInfo info;
    GetFontInfo(&info);  // Mac Font Manager call
    return info.ascent + info.leading;
}

// Internal: Calculate line spacing (ascent + descent + leading)
static short _get_font_line_spacing(TextSpec *font) {
    FontInfo info;
    GetFontInfo(&info);
    return info.ascent + info.descent + info.leading;
}
```

#### Justification Flags

```c
// From screen_drawing.h:57-65
enum { /* justification flags for _draw_screen_text */
    _no_flags           = 0x00,
    _center_horizontal  = 0x01,  // Center text horizontally in rect
    _center_vertical    = 0x02,  // Center text vertically in rect
    _right_justified    = 0x04,  // Align right edge
    _top_justified      = 0x08,  // Align to top
    _bottom_justified   = 0x10,  // Align to bottom
    _wrap_text          = 0x20   // Auto-wrap at word boundaries
};
```

#### Color System

**Screen Colors** (loaded from 'clut' resource ID 130):

```c
// From screen_drawing.h:34-55
enum {
    _energy_weapon_full_color,           // 0 - Fusion bar full
    _energy_weapon_empty_color,          // 1 - Fusion bar empty
    _black_color,                        // 2 - Pure black
    _inventory_text_color,               // 3 - Item names
    _inventory_header_background_color,  // 4 - Header bg
    _inventory_background_color,         // 5 - List background
    PLAYER_COLOR_BASE_INDEX,             // 6 - Start of 8 player colors (6-13)

    _white_color = 14,                   // 14 - Pure white
    _invalid_weapon_color,               // 15 - Grayed out
    _computer_border_background_text_color, // 16
    _computer_border_text_color,         // 17
    _computer_interface_text_color,      // 18 - Terminal body
    _computer_interface_color_purple,    // 19
    _computer_interface_color_red,       // 20
    // ... more terminal colors (21-25)
};

// Get color from palette
void _get_interface_color(short color_index, RGBColor *color) {
    *color = (*screen_colors)->ctTable[color_index].rgb;
}
```

#### Usage by System

| System | Fonts Used | Purpose |
|--------|------------|---------|
| HUD (`game_window.c`) | 1, 3 | Weapon name, ammo count |
| Terminals (`computer_interface.c`) | 4, 5 | Body text, titles |
| Overhead Map (`overhead_map.c`) | 0 | Level annotations |
| Network (`network_dialogs.c`) | 2, 6 | Player names, stats |
| Inventory | 0, 3 | Item names, counts |

#### Interface Rectangles

The font system uses predefined screen regions for text placement:

```c
// From screen_drawing.h:8-30
enum {
    /* game window rectangles */
    _player_name_rect = 0,    // Player name display
    _oxygen_rect,             // O2 bar area
    _shield_rect,             // Shield bar area
    _motion_sensor_rect,      // Radar display
    _microphone_rect,         // Voice indicator
    _inventory_rect,          // Item list
    _weapon_display_rect,     // Weapon name/ammo

    /* interface rectangles */
    _new_game_button_rect,    // Main menu buttons...
    _load_game_button_rect,
    // ... more button rects
    NUMBER_OF_INTERFACE_RECTANGLES
};

// Get rectangle by ID
screen_rectangle *get_interface_rectangle(short index);
```

Rectangles are loaded from 'nrct' resource ID 128 at initialization.

#### Porting Considerations

The font system is **entirely Mac-specific**, using:
- QuickDraw: `TextFont()`, `TextSize()`, `DrawText()`, `TruncText()`, `CharWidth()`, `TextWidth()`
- Mac Font Manager: `GetFontInfo()` for metrics
- Resource Manager: Load font specs from `finf` resource ID 128, colors from `clut` resource 130

**Porting Options**:
1. **Bitmap font renderer**: Extract Marathon fonts as sprite sheets, render character-by-character
2. **stb_truetype**: Load TrueType fonts for modern rendering, implement word wrap yourself
3. **Aleph One approach**: Use SDL_ttf with bundled fonts, wrap their API
4. **Simple option**: Use a single monospace bitmap font (8x16 pixels per character)

**Key Functions to Replace**:
- `_draw_screen_text()` → Custom text renderer with your font system
- `_text_width()` → Calculate string width from glyph widths
- `_get_font_line_height()` → Return fixed line height for your font
- `_get_interface_color()` → Map to your 32-bit ARGB colors

**Minimal Implementation** (for porting):
```c
// Simple fixed-width font replacement
void _draw_screen_text_portable(char *text, screen_rectangle *dest,
                                 short flags, short font_id, short color) {
    int char_width = 8, char_height = 16;  // Fixed-width font
    int x = dest->left, y = dest->top;

    if (flags & _center_horizontal) {
        x = dest->left + (RECTANGLE_WIDTH(dest) - strlen(text) * char_width) / 2;
    }
    if (flags & _center_vertical) {
        y = dest->top + (RECTANGLE_HEIGHT(dest) - char_height) / 2;
    }

    uint32_t argb_color = convert_index_to_argb(color);
    for (int i = 0; text[i]; i++) {
        draw_glyph(text[i], x + i * char_width, y, argb_color);
    }
}
```

---

## 22. Screen Effects & Fades

Marathon uses color table manipulation for various visual effects including damage feedback, teleportation, and environmental effects.

### Fade Types

```c
enum /* fade types */ {
    // Cinematic fades
    _start_cinematic_fade_in,
    _cinematic_fade_in,
    _long_cinematic_fade_in,
    _cinematic_fade_out,
    _end_cinematic_fade_out,

    // Damage flashes (brief, intense)
    _fade_red,           // Standard damage
    _fade_big_red,       // Heavy damage
    _fade_bonus,         // Powerup pickup
    _fade_bright,        // Flash effect
    _fade_long_bright,   // Extended flash
    _fade_yellow,        // Electrical damage
    _fade_big_yellow,    // Heavy electrical
    _fade_purple,        // Fusion damage
    _fade_cyan,          // Compiler damage
    _fade_white,         // Explosion flash
    _fade_big_white,     // Nuclear flash
    _fade_orange,        // Fire damage
    _fade_long_orange,   // Burning
    _fade_green,         // Acid/goo damage
    _fade_long_green,    // Prolonged acid

    // Special effects
    _fade_static,        // TV static effect
    _fade_negative,      // Photo negative
    _fade_big_negative,  // Intense negative
    _fade_flicker_negative, // Flickering negative

    // Blend effects (dodge/burn)
    _fade_dodge_purple,  // Additive purple
    _fade_burn_cyan,     // Subtractive cyan
    _fade_dodge_yellow,  // Additive yellow
    _fade_burn_green,    // Subtractive green

    // Environmental tints (persistent)
    _fade_tint_green,    // In goo/sewage
    _fade_tint_blue,     // Underwater
    _fade_tint_orange,   // In lava (briefly!)
    _fade_tint_gross,    // Sewage (brown-green)

    NUMBER_OF_FADE_TYPES
};
```

### Fade Implementation

**Color Table Modification**:
```c
// Fades work by modifying the color lookup table
void apply_fade_to_color_table(
    struct rgb_color *original_colors,  // Base palette
    struct rgb_color *faded_colors,     // Output palette
    short fade_type,
    fixed intensity)                    // 0 = no effect, FIXED_ONE = full
{
    struct fade_definition *fade = &fade_definitions[fade_type];

    for (int i = 0; i < 256; i++) {
        // Blend original color toward fade color
        faded_colors[i].red = blend(original_colors[i].red,
                                    fade->color.red, intensity);
        faded_colors[i].green = blend(original_colors[i].green,
                                      fade->color.green, intensity);
        faded_colors[i].blue = blend(original_colors[i].blue,
                                     fade->color.blue, intensity);
    }
}
```

### Damage Flash Sequence

```
Player takes damage:
    │
    ├─► Determine damage type
    │     └─► Select appropriate fade color
    │
    ├─► Start flash at maximum intensity
    │     └─► intensity = FIXED_ONE
    │
    └─► Decay over time
          │
          Tick 0:  ████████████████  (100%)
          Tick 1:  ██████████████    (87%)
          Tick 2:  ████████████      (75%)
          Tick 3:  ██████████        (62%)
          Tick 4:  ████████          (50%)
          Tick 5:  ██████            (37%)
          Tick 6:  ████              (25%)
          Tick 7:  ██                (12%)
          Tick 8:                    (0% - done)
```

### Damage Type to Fade Mapping

| Damage Type | Fade Effect | Color |
|-------------|-------------|-------|
| `_damage_explosion` | `_fade_red` | Red flash |
| `_damage_projectile` | `_fade_red` | Red flash |
| `_damage_electrical_staff` | `_fade_yellow` | Yellow flash |
| `_damage_fusion_bolt` | `_fade_purple` | Purple flash |
| `_damage_compiler_bolt` | `_fade_cyan` | Cyan flash |
| `_damage_flame` | `_fade_orange` | Orange flash |
| `_damage_lava` | `_fade_long_orange` | Sustained orange |
| `_damage_goo` | `_fade_green` | Green flash |
| `_damage_suffocation` | `_fade_long_bright` | White fade |
| Pickup bonus | `_fade_bonus` | Brief white flash |

### Environmental Tints

**Underwater Effects**:
```c
// Applied when player head is below media surface
void update_media_tint(short media_type) {
    switch (media_type) {
        case _media_water:
            set_fade_effect(_fade_tint_blue);
            break;
        case _media_lava:
            set_fade_effect(_fade_tint_orange);
            break;
        case _media_goo:
        case _media_sewage:
            set_fade_effect(_fade_tint_green);
            break;
        case _media_jjaro:
            set_fade_effect(_fade_tint_gross);
            break;
    }
}
```

**Tint Visualization**:
```
Normal view:                 Underwater (blue tint):
┌─────────────────┐         ┌─────────────────┐
│ Full color      │         │ Blue-shifted    │
│ spectrum        │         │ Reduced red     │
│ visible         │         │ Enhanced blue   │
└─────────────────┘         └─────────────────┘

Lava (orange tint):          Goo (green tint):
┌─────────────────┐         ┌─────────────────┐
│ Orange-shifted  │         │ Green-shifted   │
│ High saturation │         │ Sickly color    │
│ DANGER!         │         │ Toxic feel      │
└─────────────────┘         └─────────────────┘
```

### Teleport Effects

**Teleport Sequence**:
```
Pre-teleport:
    1. _cinematic_fade_out starts
    2. Screen fades to black over ~0.5 seconds
    3. Player teleported to destination

Post-teleport:
    1. _cinematic_fade_in starts
    2. Screen fades from black over ~0.5 seconds
    3. Normal view restored
```

### Gamma Correction

```c
#define NUMBER_OF_GAMMA_LEVELS 8

// User-adjustable brightness
// Each gamma level has pre-computed color tables
// Higher gamma = brighter midtones
```

**Gamma Curve**:
```
Input → Output brightness

Gamma 0 (darkest):    ──────────╱
Gamma 4 (default):   ────────╱
Gamma 7 (brightest): ──────╱

Each level shifts the brightness curve upward
```

---

## 23. View Bobbing & Camera System

Marathon's camera system creates a sense of physicality through view bobbing, weapon sway, and height adjustments.

### Step Cycle Variables

```c
// In physics_variables structure:
fixed step_phase;      // Position in walk cycle [0, FIXED_ONE)
fixed step_amplitude;  // Intensity based on velocity [0, FIXED_ONE)

// In physics constants:
fixed step_delta;      // Phase increment per tick
fixed camera_height;   // Base eye level above feet
```

### View Bob Calculation

```c
// From physics.c - camera height adjustment
world_distance calculate_step_height(struct physics_variables *variables) {
    fixed phase_angle = variables->step_phase >> (FIXED_FRACTIONAL_BITS - ANGULAR_BITS + 1);

    // Sine wave based on step phase
    fixed raw_step = (constants->step_amplitude * sine_table[phase_angle]) >> TRIG_SHIFT;

    // Scale by current amplitude (velocity-based)
    fixed step_height = (raw_step * variables->step_amplitude) >> FIXED_FRACTIONAL_BITS;

    return FIXED_TO_WORLD(step_height);
}

// Apply to camera
player->camera_location.z = player->location.z +
                            constants->camera_height +
                            calculate_step_height(variables);
```

### Step Phase Cycle

**Phase Progression**:
```
Walking cycle (phase 0 → FIXED_ONE → wraps):

Phase:     0        16384     32768     49152     65536
           ↓          ↓         ↓         ↓         ↓
Sine:      0        +1.0       0        -1.0       0
           ↓          ↓         ↓         ↓         ↓
Camera:  Center    Highest   Center    Lowest   Center

          ┌──○──┐           ┌──●──┐           ┌──○──┐
        ╱        ╲        ╱        ╲        ╱        ╲
      ╱            ╲    ╱            ╲    ╱            ╲
    ●                ●●                ○○                ●
  Left foot       Both feet        Right foot       Both feet
  down            center           down             center
```

### Amplitude Based on Velocity

```c
// step_amplitude scales with movement speed
void update_step_amplitude(struct physics_variables *variables) {
    fixed speed = isqrt(variables->velocity * variables->velocity +
                        variables->perpendicular_velocity *
                        variables->perpendicular_velocity);

    // Faster movement = more pronounced bob
    variables->step_amplitude = MIN(speed * BOB_SCALE, FIXED_ONE);
}
```

**Amplitude Examples**:
```
Standing still:   step_amplitude = 0        → No bob
Walking slowly:   step_amplitude = 0.3      → Gentle bob
Running:          step_amplitude = 0.8      → Strong bob
Max speed:        step_amplitude = 1.0      → Full bob effect
```

### Weapon Bob Synchronization

**Weapon uses same step variables**:
```c
// Weapon bob mirrors camera bob
struct weapon_display_information {
    fixed vertical_position;    // Affected by step_phase
    fixed horizontal_position;  // Slight horizontal sway
};

// Weapon moves opposite to camera for stability illusion
weapon.vertical_offset = -step_height / 2;  // Counters camera bob
weapon.horizontal_offset = (step_phase_shifted) / 4;  // Side sway
```

**Combined Effect**:
```
                    Camera bobs UP
                         ↑
    ┌─────────────────────────────────────┐
    │                                     │
    │              ╱╲                     │
    │             ╱  ╲                    │
    │            ╱    ╲   World moves up  │
    │           ╱      ╲                  │
    │          ╱        ╲                 │
    ├─────────────────────────────────────┤
    │      ┌─────────┐                    │
    │      │ WEAPON  │ ← Weapon bobs DOWN │
    │      │ (counters camera bob)        │
    │      └─────────┘                    │
    └─────────────────────────────────────┘

Result: Weapon appears more stable while world bounces
```

### Camera Height States

```c
// Different heights for different states
camera_height (standing):    ~819 (0.8 WORLD_ONE)
camera_height (crouching):   ~512 (0.5 WORLD_ONE)  // If implemented
dead_height:                 ~256 (0.25 WORLD_ONE) // Fallen viewpoint
```

### Landing Impact

**Fall landing causes view dip**:
```c
if (just_landed && fall_velocity > threshold) {
    // Temporary downward camera offset
    landing_offset = -fall_velocity * LANDING_SCALE;

    // Recovers over several ticks
    landing_recovery_rate = -landing_offset / RECOVERY_TICKS;
}
```

**Landing Sequence**:
```
Frame:     Landing  +1    +2    +3    +4    +5    Normal
Camera:    ▼▼▼▼▼▼   ▼▼▼▼  ▼▼▼   ▼▼    ▼     ─     ─

           Sudden drop from impact, gradual recovery
```

### Pitch and Look

**View pitch** (looking up/down):
```c
struct physics_variables {
    fixed elevation;           // Pitch angle
    fixed vertical_angular_velocity;  // Pitch rate
};

// Pitch affects projection
dtanpitch = world_to_screen_y * tan(elevation);
// This shifts all vertical coordinates
```

**Pitch Range**:
```
Looking up (elevation > 0):
    ┌─────────────────┐
    │   ╱ Sky visible │
    │  ╱              │
    │ ╱               │
    │@                │
    └─────────────────┘

Level view (elevation = 0):
    ┌─────────────────┐
    │                 │
    │────@────────────│
    │                 │
    └─────────────────┘

Looking down (elevation < 0):
    ┌─────────────────┐
    │                 │
    │ ╲               │
    │  ╲ Floor visible│
    │   ╲@            │
    └─────────────────┘
```

### Swimming Effects

**Underwater camera modifications**:
```c
if (player_head_below_media) {
    // Slower movement underwater
    step_delta_modifier = 0.5;

    // Reduced bob amplitude
    step_amplitude_modifier = 0.7;

    // Optional: subtle wave distortion could be added
}
```

---

## 24. cseries.lib Utility Library

> **🔧 For Porting:** Most of cseries.lib is portable! Key actions:
> - Keep: `cseries.h` (types, macros), `byte_swapping.c`, `rle.c`, `checksum.c`
> - Replace: `macintosh_cseries.h` Mac includes with standard headers
> - Replace: Memory functions (`NewPtr`→`malloc`, `NewHandle`→`malloc`, `HLock`/`HUnlock`→remove)
> - Replace: `TickCount()` → your timing function (milliseconds/60)

The `cseries.lib` directory (~4,400 lines) provides Marathon's foundational utility layer. Originally developed for earlier Bungie projects (Minotaur, Pathways), it abstracts platform differences and provides common functionality used throughout the engine.

### 24.1 Directory Structure

```
cseries.lib/
├── cseries.h              # Core types and macros (platform-independent)
├── macintosh_cseries.h    # Mac-specific extensions
├── textures.h             # Pixel and bitmap definitions
├── byte_swapping.h/.c     # Endianness handling
├── rle.h/.c               # Run-length encoding compression
├── checksum.h/.c          # CRC and checksum algorithms
├── mytm.h/.c              # Time Manager abstraction
├── preferences.h/.c       # Preferences file handling
├── macintosh_utilities.c  # Mac Toolbox wrappers
├── dialogs.c              # Dialog utilities
├── devices.c              # Display device management
└── cseries.a              # 68K assembly optimizations
```

### 24.2 cseries.h - Core Foundation

The heart of the utility library, providing platform-independent definitions:

**Basic Types**:
```c
typedef unsigned short word;       // 16-bit unsigned
typedef unsigned char byte;        // 8-bit unsigned
typedef byte boolean;              // TRUE/FALSE
typedef long fixed;                // 16.16 fixed-point
typedef void *handle;              // Relocatable memory pointer
```

**Limit Constants**:
```c
enum {
    UNSIGNED_LONG_MAX  = 4294967295,
    LONG_MAX           = 2147483647L,
    LONG_MIN           = (-2147483648L),
    SHORT_MAX          = 32767,
    SHORT_MIN          = (-32768),
    UNSIGNED_CHAR_MAX  = 255
};
```

**Essential Macros**:
```c
#define TRUE   1
#define FALSE  0
#define NONE   -1                  // Invalid index marker

#define KILO   1024
#define MEG    (KILO*KILO)

#define MACHINE_TICKS_PER_SECOND 60

// Math utilities
#define SGN(x)    ((x)?((x)<0?-1:1):0)
#define ABS(x)    ((x>=0) ? (x) : -(x))
#define MIN(a,b)  ((a)>(b)?(b):(a))
#define MAX(a,b)  ((a)>(b)?(a):(b))

// Value clamping
#define FLOOR(n,floor)       ((n)<(floor)?(floor):(n))
#define CEILING(n,ceiling)   ((n)>(ceiling)?(ceiling):(n))
#define PIN(n,floor,ceiling) ((n)<(floor) ? (floor) : CEILING(n,ceiling))

// XOR swap (in-place without temporary)
#define SWAP(a,b)  a^= b, b^= a, a^= b

// Bit flag operations
#define FLAG(b)              (1<<(b))
#define TEST_FLAG16(f, b)    ((f)&(word)FLAG(b))
#define SET_FLAG16(f, b, v)  ((v) ? ((f)|=(word)FLAG(b)) : ((f)&=(word)~FLAG(b)))
#define TEST_FLAG32(f, b)    ((f)&(unsigned long)FLAG(b))
#define SET_FLAG32(f, b, v)  ((v) ? ((f)|=(unsigned long)FLAG(b)) : ((f)&=(unsigned long)~FLAG(b)))
```

**Fixed-Point Mathematics**:
```c
#define FIXED_FRACTIONAL_BITS  16
#define FIXED_ONE              ((fixed)(1<<FIXED_FRACTIONAL_BITS))  // 65536
#define FIXED_ONE_HALF         ((fixed)(1<<(FIXED_FRACTIONAL_BITS-1)))  // 32768

// Conversions
#define INTEGER_TO_FIXED(s)        (((fixed)(s))<<FIXED_FRACTIONAL_BITS)
#define FIXED_TO_INTEGER(f)        ((short)((f)>>FIXED_FRACTIONAL_BITS))
#define FIXED_TO_INTEGER_ROUND(f)  FIXED_TO_INTEGER((f)+FIXED_ONE_HALF)
#define FIXED_FRACTIONAL_PART(f)   (((fixed)(f))&(FIXED_ONE-1))

// Float interop (rarely used)
#define FIXED_TO_FLOAT(f)  (((double)(f))/FIXED_ONE)
#define FLOAT_TO_FIXED(f)  ((fixed)((f)*FIXED_ONE))
```

**16.16 Fixed-Point Format**:
```
┌─────────────────────────────────────────────────────────────────┐
│ 31                    16 15                                   0 │
├─────────────────────────┼───────────────────────────────────────┤
│   INTEGER PART (signed) │         FRACTIONAL PART               │
│      16 bits            │            16 bits                    │
├─────────────────────────┼───────────────────────────────────────┤
│ Range: -32768 to 32767  │ Precision: 1/65536 ≈ 0.0000153        │
└─────────────────────────┴───────────────────────────────────────┘

Examples:
  1.0     = 0x00010000 = 65536
  0.5     = 0x00008000 = 32768
  -1.0    = 0xFFFF0000 = -65536
  3.14159 ≈ 0x0003243F = 205887
```

**Debug Macros**:
```c
#ifdef DEBUG
    #define halt()            _assertion_failure(NULL, __FILE__, __LINE__, TRUE)
    #define vhalt(diag)       _assertion_failure(diag, __FILE__, __LINE__, TRUE)
    #define assert(expr)      if (!(expr)) _assertion_failure(#expr, __FILE__, __LINE__, TRUE)
    #define vassert(expr,diag) if (!(expr)) _assertion_failure(diag, __FILE__, __LINE__, TRUE)
    #define warn(expr)        if (!(expr)) _assertion_failure(#expr, __FILE__, __LINE__, FALSE)
    #define pause()           _assertion_failure(NULL, __FILE__, __LINE__, FALSE)
#else
    // All macros become no-ops in release builds
    #define halt()
    #define assert(expr)
    // ...etc
#endif
```

**Memory Management**:
```c
// Override standard malloc/free with tracked versions
#define malloc(size) new_pointer(size)
#define free(ptr)    dispose_pointer(ptr)

void *new_pointer(long size);
void dispose_pointer(void *pointer);
```

### 24.3 textures.h - Pixel Types

Defines pixel formats and bitmap structures used throughout rendering:

**Pixel Types**:
```c
typedef unsigned char pixel8;    // 8-bit indexed color
typedef unsigned short pixel16;  // 16-bit RGB (5-5-5)
typedef unsigned long pixel32;   // 32-bit RGB (8-8-8)

#define PIXEL8_MAXIMUM_COLORS   256
#define PIXEL16_MAXIMUM_COLORS  32768
#define PIXEL32_MAXIMUM_COLORS  16777216
```

**16-bit Pixel Macros (RGB 5-5-5)**:
```c
#define PIXEL16_BITS             5
#define PIXEL16_MAXIMUM_COMPONENT 0x1f  // 31

#define RED16(p)    ((p)>>10)
#define GREEN16(p)  (((p)>>5)&PIXEL16_MAXIMUM_COMPONENT)
#define BLUE16(p)   ((p)&PIXEL16_MAXIMUM_COMPONENT)
#define BUILD_PIXEL16(r,g,b) (((r)<<10)|((g)<<5)|(b))

// Convert from 16-bit Mac RGB to pixel16
#define RGBCOLOR_TO_PIXEL16(r,g,b) \
    (((pixel16)((r)>>1)&0x7c00)|((pixel16)((g)>>6)&0x03e0)|((pixel16)((b)>>11)&0x1f))
```

**32-bit Pixel Macros (RGB 8-8-8)**:
```c
#define PIXEL32_BITS              8
#define PIXEL32_MAXIMUM_COMPONENT 0xff  // 255

#define RED32(p)    ((p)>>16)
#define GREEN32(p)  (((p)>>8)&PIXEL32_MAXIMUM_COMPONENT)
#define BLUE32(p)   ((p)&PIXEL32_MAXIMUM_COMPONENT)
#define BUILD_PIXEL32(r,g,b) (((r)<<16)|((g)<<8)|(b))
```

**Pixel Format Diagrams**:
```
pixel16 (RGB 5-5-5, 15-bit color):
┌───┬─────────────┬─────────────┬─────────────┐
│ X │    RED      │   GREEN     │    BLUE     │
│ 1 │   5 bits    │   5 bits    │   5 bits    │
├───┼─────────────┼─────────────┼─────────────┤
│15 │ 14      10  │  9       5  │  4       0  │
└───┴─────────────┴─────────────┴─────────────┘
  Total: 32,768 colors (bit 15 unused)

pixel32 (RGB 8-8-8, 24-bit color):
┌─────────────┬─────────────┬─────────────┬─────────────┐
│   (unused)  │    RED      │   GREEN     │    BLUE     │
│   8 bits    │   8 bits    │   8 bits    │   8 bits    │
├─────────────┼─────────────┼─────────────┼─────────────┤
│ 31      24  │ 23      16  │ 15       8  │  7       0  │
└─────────────┴─────────────┴─────────────┴─────────────┘
  Total: 16,777,216 colors (alpha channel unused)

pixel8 (Indexed, 8-bit):
┌─────────────────────────────────┐
│       PALETTE INDEX             │
│          8 bits                 │
├─────────────────────────────────┤
│  7                           0  │
└─────────────────────────────────┘
  Points to color_table entry (256 colors)
```

**Color Table Structure**:
```c
struct rgb_color {
    word red, green, blue;   // 16-bit components (0-65535)
};

struct color_table {
    short color_count;
    struct rgb_color colors[256];
};
```

**Bitmap Definition**:
```c
enum {  // bitmap flags
    _COLUMN_ORDER_BIT = 0x8000,  // Data stored column-major (for walls)
    _TRANSPARENT_BIT  = 0x4000   // Has transparent pixels
};

struct bitmap_definition {
    short width, height;      // Dimensions in pixels
    short bytes_per_row;      // NONE = RLE compressed
    short flags;              // _COLUMN_ORDER_BIT, _TRANSPARENT_BIT
    short bit_depth;          // Always 8 for indexed color
    short unused[8];
    pixel8 *row_addresses[1]; // Flexible array of row pointers
};
```

**Bitmap Memory Layout**:
```
Row-Order Bitmap (floors/ceilings):      Column-Order Bitmap (walls):
┌─────────────────────────┐              ┌─────────────────────────┐
│ bitmap_definition header│              │ bitmap_definition header│
├─────────────────────────┤              ├─────────────────────────┤
│ row_addresses[0] ───────┼─┐            │ row_addresses[0] ───────┼─┐
│ row_addresses[1] ───────┼─┼─┐          │ row_addresses[1] ───────┼─┼─┐
│ row_addresses[2] ───────┼─┼─┼─┐        │        ...              │ │ │
│        ...              │ │ │ │        ├─────────────────────────┤ │ │
├─────────────────────────┤ │ │ │        │ Column 0 (all rows)     │←┘ │
│ Row 0: pixels 0..width  │←┘ │ │        │ Column 1 (all rows)     │←──┘
│ Row 1: pixels 0..width  │←──┘ │        │ Column 2 (all rows)     │
│ Row 2: pixels 0..width  │←────┘        │        ...              │
│        ...              │              └─────────────────────────┘
└─────────────────────────┘
  _COLUMN_ORDER_BIT = 0                    _COLUMN_ORDER_BIT = 1

Column-order enables efficient vertical texture mapping for walls
(reads consecutive memory addresses when drawing vertical spans)
```

### 24.4 RLE Compression (rle.c)

Run-length encoding for sprite/texture compression:

**Encoding Format**:
```
First 4 bytes: Uncompressed size (long)
Then opcodes:
  0 ≤ n < 128   : Repeat next byte (n+3) times
  128 ≤ n ≤ 255 : Copy next (n-127) bytes literally
```

**API**:
```c
// Get uncompressed size from compressed data
long get_destination_size(byte *compressed);

// Compress data, returns compressed size or -1 if expansion
long compress_bytes(byte *raw, long raw_size,
                   byte *compressed, long maximum_compressed_size);

// Decompress (caller must allocate destination)
void uncompress_bytes(byte *compressed, byte *raw);
```

**Compression Algorithm**:
```c
// Simplified logic
while (data_remaining) {
    if (next_3_bytes_are_same) {
        // Encode run: count up to 130 repeats
        run_length = count_repeats();  // 3-130
        *output++ = run_length - 3;    // 0-127
        *output++ = repeated_byte;
    } else {
        // Encode literal run: up to 128 bytes
        literal_count = count_non_repeating();  // 1-128
        *output++ = literal_count + 127;        // 128-255
        memcpy(output, source, literal_count);
    }
}
```

**RLE Encoding Visualization**:
```
Original Data (16 bytes):
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 41 │ 42 │ 43 │ FF │ FF │ FF │ FF │ FF │ FF │ FF │ 10 │ 20 │ 30 │ 30 │ 30 │ 30 │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  A    B    C   ←── 7 repeated FFs ──→              ←─ 4 repeated 30s ─→

Compressed (12 bytes):
┌──────────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 00000010 │ 83 │ 41 │ 42 │ 43 │ 04 │ FF │ 83 │ 10 │ 20 │ 01 │ 30 │
│(size=16) │lit │ A  │ B  │ C  │run │byte│lit │    │    │run │byte│
└──────────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
     ↑        ↑                   ↑         ↑              ↑
  4-byte   130=3     7 repeats   130=3    4 repeats
  header   literals  (4+3=7)     literals (1+3=4)
           (3 bytes)             (2 bytes)

Opcode interpretation:
  0-127:   Run of (n+3) copies of next byte
  128-255: Literal run of (n-127) bytes following
```

### 24.5 Byte Swapping (byte_swapping.c)

Handles endianness conversion between big-endian (Mac) and little-endian (x86) data:

**Swap Macros**:
```c
#define SWAP2(q) (((q)>>8) | (((q)<<8)&0xff00))

#define SWAP4(q) (((q)>>24) | \
                  (((q)>>8)&0xff00) | \
                  (((q)<<8)&0x00ff00) | \
                  (((q)<<24)&0xff000000))
```

**Field Descriptors**:
```c
enum {
    // Positive = skip N bytes unchanged
    // Negative = swap |N| bytes
    _byte  = 1,    // Skip 1 byte
    _2byte = -2,   // Swap 2-byte value
    _4byte = -4    // Swap 4-byte value
};

typedef short _bs_field;
```

**Byte Order Visualization**:
```
Big-Endian (Mac/Network)         Little-Endian (x86/ARM)
Most significant byte first      Least significant byte first

16-bit value 0x1234:
┌──────┬──────┐                  ┌──────┬──────┐
│  12  │  34  │     SWAP2 →      │  34  │  12  │
└──────┴──────┘                  └──────┴──────┘
addr:  N   N+1                   addr:  N   N+1

32-bit value 0x12345678:
┌──────┬──────┬──────┬──────┐    ┌──────┬──────┬──────┬──────┐
│  12  │  34  │  56  │  78  │ →  │  78  │  56  │  34  │  12  │
└──────┴──────┴──────┴──────┘    └──────┴──────┴──────┴──────┘
addr:  N   N+1  N+2  N+3         addr:  N   N+1  N+2  N+3
```

**Field Descriptor Pattern**:
```
Structure with mixed field sizes:
struct example {
    short a;      // 2 bytes - needs swap
    byte  b;      // 1 byte  - no swap
    byte  c;      // 1 byte  - no swap
    long  d;      // 4 bytes - needs swap
};

Field descriptor array:
┌────────┬────────┬────────┬────────┬────────┐
│ _2byte │ _byte  │ _byte  │ _4byte │   0    │
│  (-2)  │  (1)   │  (1)   │  (-4)  │ (end)  │
└────────┴────────┴────────┴────────┴────────┘
   swap    skip    skip     swap    terminator
```

**Platform-Conditional API**:
```c
#ifdef mac
    // Mac is native big-endian, no-op
    #define byte_swap_data(data, size, nmemb, fields)
    #define byte_swap_memory(data, type, nmemb)
#endif

#ifdef win
    // Windows needs actual swapping
    void byte_swap_data(void *data, long size, long nmemb, _bs_field *fields);
    void byte_swap_memory(void *data, _bs_field type, long nmemb);
#endif
```

**Usage Example**:
```c
// Swap an array of polygon_data structures
_bs_field polygon_fields[] = {
    _2byte,  // type (short)
    _2byte,  // flags (word)
    _2byte,  // permutation (short)
    // ... describe each field
    0        // Terminator
};

byte_swap_data(polygons, sizeof(polygon_data), polygon_count, polygon_fields);
```

### 24.6 Checksum (checksum.c)

Multiple checksum algorithms for data integrity:

**Supported Algorithms**:
```c
enum {
    ADD_CHECKSUM,       // Simple additive checksum
    FLETCHER_CHECKSUM,  // Fletcher-16 (used for networking)
    CRC32_CHECKSUM      // CRC-32 (used for file verification)
};
```

**Checksum Structure**:
```c
typedef struct {
    long bogus1;           // Obfuscation padding
    word checksum_type;    // Algorithm identifier
    union {
        word add_checksum;
        word fletcher_checksum;
        long crc32_checksum;
    } value;
    long bogus2;           // Obfuscation padding
} Checksum;
```

**API**:
```c
// Initialize a new checksum
void new_checksum(Checksum *check, word type);

// Update checksum with more data
void update_checksum(Checksum *check, word *src, long length);

// Compare two checksums
boolean equal_checksums(Checksum *check1, Checksum *check2);
```

### 24.7 Time Manager (mytm.c)

Mac Time Manager abstraction for periodic callbacks:

**Task Structure**:
```c
struct myTMTask {
    TMTask tmTask;           // Mac Time Manager task
#ifdef env68k
    long a5;                 // A5 world for 68K
#endif
    long period;             // Microseconds between calls
    myTMTaskProcPtr procedure;  // Callback function
    boolean active;
    boolean useExtendedTM;   // Use extended Time Manager
};

typedef boolean (*myTMTaskProcPtr)(void);
```

**API**:
```c
// Setup periodic task (returns task pointer)
#define myTMSetup(period, proc)   myTimeManagerSetup(period, proc, FALSE)
#define myXTMSetup(period, proc)  myTimeManagerSetup(period, proc, TRUE)

myTMTaskPtr myTimeManagerSetup(long period, myTMTaskProcPtr procedure,
                                boolean useExtendedTM);

// Reset task timer
void myTMReset(myTMTaskPtr myTask);

// Remove and cleanup task (returns NULL)
myTMTaskPtr myTMRemove(myTMTaskPtr myTask);
```

### 24.8 macintosh_cseries.h - Mac Extensions

Platform-specific extensions for Macintosh development:

**Key Constants**:
```c
#define mac  // Platform identifier

#define MACINTOSH_TICKS_PER_SECOND 60

// Key codes
#define kUP_ARROW    0x1e
#define kDOWN_ARROW  0x1f
#define kLEFT_ARROW  0x1c
#define kRIGHT_ARROW 0x1d
#define kENTER       0x03
#define kRETURN      0x0d
#define kTAB         0x09
#define kESCAPE      0x1b
#define kDELETE      0x08

// Function keys (F1-F12)
#define kcF1  0x7a
#define kcF2  0x78
// ...etc
```

**Rectangle Macros**:
```c
#define RECTANGLE_WIDTH(r)   ((r)->right-(r)->left)
#define RECTANGLE_HEIGHT(r)  ((r)->bottom-(r)->top)
#define UPPER_LEFT_CORNER(r)  ((Point *)r)
#define LOWER_RIGHT_CORNER(r) (((Point *)r)+1)
```

**System Colors**:
```c
enum {
    windowLowlight = 0,
    window33Percent,
    window66Percent,
    windowHighlight,
    highlightColor,
    gray15Percent,
    gray33Percent,
    // ... more grays
    activeAppleGreen,
    activeAppleYellow,
    // ... Apple colors
    NUMBER_OF_SYSTEM_COLORS
};

extern RGBColor system_colors[];
```

**68K Assembly Inlines**:
```c
#ifdef env68k
    #pragma parameter __D0 get_a5
    long get_a5(void)= {0x200d};           // MOVE.L A5,D0

    #pragma parameter __D0 set_a5(__D1)
    long set_a5(long a5)= {0x200d, 0x2a41}; // MOVE.L A5,D0; MOVEA.L D1,A5
#endif
```

### 24.9 Porting Considerations

When porting Marathon, cseries.lib requires these adaptations:

**Replacement Table**:
| Original | Modern Replacement |
|----------|-------------------|
| `new_pointer()` / `dispose_pointer()` | Standard `malloc()` / `free()` |
| `machine_tick_count()` | `SDL_GetTicks()` or platform equivalent |
| `myTMSetup()` | `SDL_AddTimer()` or thread-based timing |
| `byte_swap_*()` | Implement for x86 (or use `#if __BYTE_ORDER__`) |
| Mac key codes | SDL or platform scancodes |
| `RGBColor` | Platform color type |

**Keep unchanged**:
- All macros in `cseries.h` (pure C, portable)
- Fixed-point math (essential for determinism)
- RLE compression/decompression (portable)
- Checksum algorithms (portable)
- `textures.h` definitions (pure C)

**Complete File Reference**:

| File | Lines | Purpose | Portable? |
|------|-------|---------|-----------|
| `cseries.h` | ~200 | Core types, macros, fixed-point math | ✓ Yes |
| `textures.h` | ~90 | Pixel types, bitmap structures | ✓ Yes |
| `rle.c` / `rle.h` | ~190 | Run-length compression | ✓ Yes |
| `checksum.c` / `checksum.h` | ~150 | CRC32, Fletcher, additive checksums | ✓ Yes |
| `byte_swapping.h` | ~40 | Swap macros (need .c for x86) | ⚠ Partial |
| `byte_swapping.c` | ~80 | Swap implementations | ✓ Yes |
| `proximity_strcmp.c/.h` | ~50 | Fuzzy string matching | ✓ Yes |
| `macintosh_cseries.h` | ~360 | Mac Toolbox types & integration | ✗ Replace |
| `macintosh_utilities.c` | ~800 | Mac UI helpers, file dialogs | ✗ Replace |
| `mytm.c` / `mytm.h` | ~120 | Mac Time Manager wrapper | ✗ Replace |
| `dialogs.c` | ~400 | Mac dialog utilities | ✗ Replace |
| `devices.c` | ~500 | Mac display device management | ✗ Replace |
| `device_dialog.c` | ~200 | Mac device selection dialog | ✗ Replace |
| `preferences.c/.h` | ~150 | Mac preferences file I/O | ✗ Replace |
| `my32bqd.c/.h` | ~100 | 32-bit QuickDraw support | ✗ Replace |
| `macintosh_interfaces.c` | ~50 | Mac toolbox includes | ✗ Replace |
| `cseries.a` | ~300 | 68K assembly optimizations | ✗ Replace |

**Portability Summary**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    cseries.lib Portability                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PORTABLE (use directly)          MAC-SPECIFIC (replace)        │
│  ━━━━━━━━━━━━━━━━━━━━━━━          ━━━━━━━━━━━━━━━━━━━━━━━       │
│  cseries.h ─────────────┐        macintosh_cseries.h            │
│  textures.h             │        macintosh_utilities.c          │
│  rle.c/h                │        macintosh_interfaces.c         │
│  checksum.c/h           │───→    mytm.c/h                       │
│  byte_swapping.h        │ Use    dialogs.c                      │
│  proximity_strcmp.c/h   │        devices.c                      │
│                         │        device_dialog.c                │
│  ~720 lines             │        preferences.c/h                │
│  (16% of library)       │        my32bqd.c/h                    │
│                         │        cseries.a                      │
│                         │                                       │
│                         │        ~3,680 lines                   │
│                         │        (84% of library)               │
└─────────────────────────────────────────────────────────────────┘

Note: The 84% "Mac-specific" code is mostly UI helpers and
system integration. Core engine code doesn't need most of it.
```

---

## 25. Media/Liquid System

The media system handles all liquid surfaces in Marathon: water, lava, goo, sewage, and Jjaro liquids. Each polygon can contain media that affects gameplay, rendering, and sound.

### 25.1 Media Types

```c
enum {  // media types
    _media_water,   // Safe, swimmable
    _media_lava,    // Damaging, glowing
    _media_goo,     // Damaging, sticky
    _media_sewage,  // Safe, murky
    _media_jjaro,   // Safe, alien
    NUMBER_OF_MEDIA_TYPES  // 5
};

#define MAXIMUM_MEDIAS_PER_MAP 16
```

**Media Properties by Type**:

| Type | Damage | Freq | Fade Effect | Sound Set |
|------|--------|------|-------------|-----------|
| Water | None | - | Blue tint | Water sounds |
| Lava | 16 pts | Every 16 ticks | Orange/red | Lava sounds |
| Goo | 8 pts | Every 8 ticks | Green tint | Lava sounds |
| Sewage | None | - | Brown tint | Sewage sounds |
| Jjaro | None | - | Brown tint | Sewage sounds |

### 25.2 Media Data Structure

```c
struct media_data {  // 32 bytes
    short type;              // _media_water, _media_lava, etc.
    word flags;              // Sound obstruction flags

    short light_index;       // Controls height animation!

    angle current_direction; // Flow direction (0-511)
    world_distance current_magnitude;  // Flow speed

    world_distance low, high;  // Height range
    world_point2d origin;      // Texture scroll origin
    world_distance height;     // Current calculated height

    fixed minimum_light_intensity;
    shape_descriptor texture;
    short transfer_mode;

    short unused[2];
};
```

### 25.3 Height Animation via Lights

**The clever trick**: Media height is controlled by light intensity!

```c
#define CALCULATE_MEDIA_HEIGHT(m) \
    ((m)->low + FIXED_INTEGERAL_PART(((m)->high-(m)->low) * get_light_intensity((m)->light_index)))
```

**Height Animation Diagram**:
```
Light intensity: 0.0                    1.0
                 ├────────────────────────┤
                 │                        │
Media height:  low                      high
                 ▼                        ▼
           ─────┴─   ←──────────────→   ──┴────
             ░░░                         ░░░░░░░
             ░░░     Water level         ░░░░░░░
           ──┬───     rises!           ───┬────
             │                            │

Benefits:
• Reuse existing light animation system
• Smooth transitions (fading, flickering)
• Rising water tied to switch/trigger lights
• No additional animation code needed
```

### 25.4 Media Sounds

```c
enum {  // media sounds
    _media_snd_feet_entering,    // Splash when walking in
    _media_snd_feet_leaving,     // Exit splash
    _media_snd_head_entering,    // Submerge sound
    _media_snd_head_leaving,     // Surface sound
    _media_snd_splashing,        // Walking through
    _media_snd_ambient_over,     // Ambient above surface
    _media_snd_ambient_under,    // Ambient below surface
    _media_snd_platform_entering,// Platform into liquid
    _media_snd_platform_leaving, // Platform out of liquid
    NUMBER_OF_MEDIA_SOUNDS       // 9
};
```

**Sound Transitions**:
```
Player Movement Through Media:

    Above Surface          Entering               Submerged
    ─────────────────     ─────────────          ─────────────
    ambient_over    →     feet_entering    →     ambient_under
                          │                       │
                          └─ splashing (walking) ─┘
                                                  │
                          head_entering ──────────┘
```

### 25.5 Media Detonation Effects

When projectiles hit media surfaces:

```c
enum {  // detonation sizes
    _small_media_detonation_effect,   // Bullet impacts
    _medium_media_detonation_effect,  // Grenades
    _large_media_detonation_effect,   // Rockets
    _large_media_emergence_effect,    // Object surfacing
    NUMBER_OF_MEDIA_DETONATION_TYPES
};
```

**Per-Media Splash Effects**:

| Media | Small | Medium | Large | Emergence |
|-------|-------|--------|-------|-----------|
| Water | water_splash | water_splash | water_splash | water_emergence |
| Lava | lava_splash | lava_splash | lava_splash | lava_emergence |
| Goo | goo_splash | goo_splash | goo_splash | goo_emergence |
| Sewage | sewage_splash | sewage_splash | sewage_splash | sewage_emergence |
| Jjaro | jjaro_splash | jjaro_splash | jjaro_splash | jjaro_emergence |

### 25.6 Media Flow (Currents)

Media can push objects with currents:

```c
// Update media origin for texture scrolling
media->origin.x = WORLD_FRACTIONAL_PART(media->origin.x +
    ((cosine_table[media->current_direction] * media->current_magnitude) >> TRIG_SHIFT));
media->origin.y = WORLD_FRACTIONAL_PART(media->origin.y +
    ((sine_table[media->current_direction] * media->current_magnitude) >> TRIG_SHIFT));
```

**Current Effect on Player** (from physics):
```
Player in media:
  external_velocity += current_magnitude / 32  (per tick)
  direction = current_direction

This creates the "push" effect in flowing water/lava.
```

### 25.7 Submerged Effects

When player's head is below media surface:

```c
short get_media_submerged_fade_effect(short media_index) {
    // Returns fade effect type for this media
    return definition->submerged_fade_effect;
}
```

**Underwater Visual Effects**:
```
┌─────────────────────────────────────────────────────────────┐
│                    ABOVE SURFACE                            │
│   Normal rendering, full visibility, normal sounds          │
├─────────────────────────────────────────────────────────────┤
│ ～～～～～～～～ SURFACE LINE ～～～～～～～～～～～～～～～  │
├─────────────────────────────────────────────────────────────┤
│                    BELOW SURFACE                            │
│   • Screen fade applied (blue/green/orange tint)            │
│   • Ambient sound changes to underwater                     │
│   • Oxygen starts depleting (if applicable)                 │
│   • Movement slowed (gravity/terminal velocity halved)      │
└─────────────────────────────────────────────────────────────┘
```

### 25.8 Media Damage

```c
struct damage_definition *get_media_damage(short media_index, fixed scale) {
    struct media_definition *definition = get_media_definition(media->type);

    // Check damage frequency mask
    if (dynamic_world->tick_count & definition->damage_frequency)
        return NULL;  // Not this tick

    return &definition->damage;
}
```

**Damage Frequency Masks**:
- Lava: `0x0F` (every 16 ticks = ~0.5 sec)
- Goo: `0x07` (every 8 ticks = ~0.25 sec)
- Others: `0x00` (no damage)

### 25.9 Media Definition Structure

```c
struct media_definition {
    short collection, shape;      // Texture source
    short shape_count;            // Texture variations
    short shape_frequency;        // Animation rate

    short transfer_mode;          // Rendering mode

    short damage_frequency;       // Tick mask for damage
    struct damage_definition damage;

    short detonation_effects[NUMBER_OF_MEDIA_DETONATION_TYPES];
    short sounds[NUMBER_OF_MEDIA_SOUNDS];

    short submerged_fade_effect;  // Screen tint when under
};
```

---

## 26. Visual Effects System

Effects are temporary visual objects (explosions, splashes, sparks, blood) that spawn, animate, and auto-remove.

### 26.1 Effect Types

```c
enum {  // 64+ effect types
    // Weapon effects
    _effect_rocket_explosion,
    _effect_rocket_contrail,
    _effect_grenade_explosion,
    _effect_grenade_contrail,
    _effect_bullet_ricochet,
    _effect_flamethrower_burst,

    // Blood effects (per monster type)
    _effect_fighter_blood_splash,
    _effect_player_blood_splash,
    _effect_civilian_blood_splash,
    _effect_enforcer_blood_splash,
    _effect_trooper_blood_splash,
    _effect_cyborg_blood_splash,

    // Energy weapon effects
    _effect_compiler_bolt_minor_detonation,
    _effect_compiler_bolt_major_detonation,
    _effect_minor_fusion_detonation,
    _effect_major_fusion_detonation,

    // Teleportation
    _effect_teleport_object_in,
    _effect_teleport_object_out,

    // Media splashes (per liquid type)
    _effect_small_water_splash,
    _effect_medium_water_splash,
    _effect_large_water_splash,
    _effect_large_water_emergence,
    // ... lava, sewage, goo, jjaro variants

    // Destructible scenery
    _effect_water_lamp_breaking,
    _effect_lava_lamp_breaking,
    _effect_sewage_lamp_breaking,
    _effect_alien_lamp_breaking,

    // Misc
    _effect_metallic_clang,
    _effect_fist_detonation,

    NUMBER_OF_EFFECT_TYPES  // ~64
};

#define MAXIMUM_EFFECTS_PER_MAP 64
```

### 26.2 Effect Data Structure

```c
struct effect_data {  // 16 bytes
    short type;          // Effect type enum
    short object_index;  // Associated map object

    word flags;          // [slot_used.1] [unused.15]

    short data;          // Extra data (e.g., twin object for teleport)
    short delay;         // Ticks before effect becomes visible

    short unused[11];
};
```

### 26.3 Effect Definition

```c
enum {  // effect flags
    _end_when_animation_loops         = 0x0001,
    _end_when_transfer_animation_loops = 0x0002,
    _sound_only                       = 0x0004,  // No visual, just sound
    _make_twin_visible                = 0x0008,  // Show linked object when done
    _media_effect                     = 0x0010   // Associated with liquid
};

struct effect_definition {
    short collection, shape;  // Visual source
    fixed sound_pitch;        // Audio pitch modifier
    word flags;               // Behavior flags
    short delay;              // Random delay range (ticks)
    short delay_sound;        // Sound when delay ends
};
```

### 26.4 Effect Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    EFFECT LIFECYCLE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  new_effect()                                                   │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────┐                                            │
│  │ Delay Phase     │  (invisible, waiting)                      │
│  │ effect->delay   │  If delay>0, randomly vary start time      │
│  └────────┬────────┘                                            │
│           │ delay reaches 0                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ Animation Phase │  (visible, animating)                      │
│  │ animate_object()│  Play through shape frames                 │
│  └────────┬────────┘                                            │
│           │ last frame reached                                  │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ remove_effect() │  Delete from world                         │
│  │ Free slot       │  Object and effect data freed              │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 26.5 Creating Effects

```c
short new_effect(
    world_point3d *origin,     // World position
    short polygon_index,       // Containing polygon
    short type,                // Effect type enum
    angle facing)              // Direction (for directional effects)
{
    struct effect_definition *definition = get_effect_definition(type);

    // Sound-only effects: just play sound, no object
    if (definition->flags & _sound_only) {
        play_world_sound(polygon_index, origin,
            get_shape_animation_data(shape)->first_frame_sound);
        return NONE;
    }

    // Create map object for visual effect
    short object_index = new_map_object3d(origin, polygon_index,
        BUILD_DESCRIPTOR(definition->collection, definition->shape), facing);

    // Set up effect tracking
    effect->type = type;
    effect->object_index = object_index;
    effect->delay = definition->delay ? random() % definition->delay : 0;

    // Initially invisible if delayed
    if (effect->delay)
        SET_OBJECT_INVISIBILITY(object, TRUE);

    return effect_index;
}
```

### 26.6 Updating Effects

```c
void update_effects(void) {  // Called every tick
    for (effect_index = 0; effect_index < MAXIMUM_EFFECTS_PER_MAP; ++effect_index) {
        if (SLOT_IS_USED(effect)) {
            if (effect->delay) {
                // Delayed effect: count down
                if (!(--effect->delay)) {
                    SET_OBJECT_INVISIBILITY(object, FALSE);
                    play_object_sound(effect->object_index, definition->delay_sound);
                }
            } else {
                // Active effect: animate
                animate_object(effect->object_index);

                // Check for termination
                if ((animation_flags & _obj_last_frame_animated) &&
                    (definition->flags & _end_when_animation_loops)) {
                    remove_effect(effect_index);

                    // Make twin visible if needed (teleport-in)
                    if (definition->flags & _make_twin_visible)
                        SET_OBJECT_INVISIBILITY(twin_object, FALSE);
                }
            }
        }
    }
}
```

### 26.7 Teleportation Effects

Special two-phase effects for object teleportation:

```c
void teleport_object_out(short object_index) {
    // Create fade-out effect at object location
    effect_index = new_effect(&object->location, object->polygon,
        _effect_teleport_object_out, object->facing);

    // Effect copies object appearance
    effect_object->shape = object->shape;
    effect_object->transfer_mode = _xfer_fold_out;  // Shrinking effect
    effect_object->transfer_period = TELEPORTING_MIDPOINT;

    // Hide original object
    SET_OBJECT_INVISIBILITY(object, TRUE);

    play_object_sound(effect->object_index, _snd_teleport_out);
}

void teleport_object_in(short object_index) {
    effect_index = new_effect(&object->location, object->polygon,
        _effect_teleport_object_in, object->facing);

    effect->data = object_index;  // Remember which object to reveal
    effect_object->transfer_mode = _xfer_fold_in;  // Growing effect

    // Object stays invisible until effect completes
    // (_make_twin_visible flag handles reveal)
}
```

**Teleport Visual**:
```
TELEPORT OUT:                    TELEPORT IN:
    ██████                           ░
   ████████                         ░░░
  ██████████   →   ████   →   ░   ░░░░░   →  ██████████
   ████████          ██            ░░░       ████████
    ██████                          ░         ██████

  Object      Shrinking    Gone   Sparkle  Growing    Object
  visible     (_fold_out)        effect   (_fold_in)  visible
```

### 26.8 Effect Categories

| Category | Examples | Behavior |
|----------|----------|----------|
| Explosions | Rocket, grenade, fusion | Single animation, sound |
| Contrails | Rocket trail, fusion trail | Spawn behind projectile |
| Blood | Per-monster type | Color-coded splashes |
| Splashes | Per-media type | Size varies by impact |
| Sparks | Hunter, defender, juggernaut | Quick flash effects |
| Teleport | In/out | Special transfer modes |
| Destruction | Lamp breaking | Triggered by damage |
| Sound-only | Fist hit, melee | No visual component |

---

## 27. Scenery Objects

Scenery consists of static decorative objects placed in the world: lights, debris, blood, machinery, and environmental details.

### 27.1 Scenery Flags

```c
enum {
    _scenery_is_solid         = 0x0001,  // Blocks movement
    _scenery_is_animated      = 0x0002,  // Has animation frames
    _scenery_can_be_destroyed = 0x0004   // Takes damage
};
```

### 27.2 Scenery Definition Structure

```c
struct scenery_definition {
    word flags;                   // Behavior flags
    shape_descriptor shape;       // Visual appearance

    world_distance radius;        // Collision radius
    world_distance height;        // Collision height (negative = hanging)

    short destroyed_effect;       // Effect when destroyed
    shape_descriptor destroyed_shape;  // Appearance after destruction
};
```

### 27.3 Scenery Types by Environment

**Lava Environment** (`_collection_scenery2`):

| Index | Description | Solid | Destructible |
|-------|-------------|-------|--------------|
| 0 | Light dirt | No | No |
| 1 | Dark dirt | No | No |
| 2-4 | Bones/skull | No | No |
| 5-6 | Hanging lights | Yes | Yes → lamp_breaking |
| 7-9 | Cylinders/blocks | Yes | No |

**Water Environment** (`_collection_scenery1`):

| Index | Description | Solid | Destructible |
|-------|-------------|-------|--------------|
| 0 | Pistol clip | No | No |
| 1-2 | Lights | Yes | Yes → lamp_breaking |
| 3 | Siren | Yes | Yes → explosion |
| 4-6 | Blood, puddles | No | No |
| 7 | Water animation | No | Animated |
| 8-9 | Supply cans | Yes | No |
| 10 | Machine | No | Animated |

**Sewage Environment** (`_collection_scenery3`):

| Index | Description | Solid | Destructible |
|-------|-------------|-------|--------------|
| 0-1 | Green lights | Yes | Yes |
| 2-5 | Junk, antennas | No | No |
| 6 | Supply can | Yes | No |
| 7-10 | Bones, gore | No | No |

**Alien Environment** (`_collection_scenery5`):

| Index | Description | Solid | Destructible |
|-------|-------------|-------|--------------|
| 0-2 | Alien lights | Yes | Yes |
| 3-8 | Organic objects | No | No |
| 9 | Hunter shield | No | No |
| 10 | Alien sludge | No | No |

### 27.4 Creating Scenery

```c
short new_scenery(
    struct object_location *location,
    short scenery_type)
{
    struct scenery_definition *definition = get_scenery_definition(scenery_type);

    // Create map object with scenery shape
    object_index = new_map_object(location, definition->shape);

    // Configure object
    SET_OBJECT_OWNER(object, _object_is_scenery);
    SET_OBJECT_SOLIDITY(object, definition->flags & _scenery_is_solid);
    object->permutation = scenery_type;  // Remember type for destruction

    return object_index;
}
```

### 27.5 Animated Scenery

```c
#define MAXIMUM_ANIMATED_SCENERY_OBJECTS 20

static short animated_scenery_object_count;
static short *animated_scenery_object_indexes;

void randomize_scenery_shapes(void) {
    animated_scenery_object_count = 0;

    for_each_object(object) {
        if (GET_OBJECT_OWNER(object) == _object_is_scenery) {
            // Try to randomize starting frame
            if (!randomize_object_sequence(object_index, definition->shape)) {
                // If animated, add to update list
                if (animated_scenery_object_count < MAXIMUM_ANIMATED_SCENERY_OBJECTS)
                    animated_scenery_object_indexes[animated_scenery_object_count++] = object_index;
            }
        }
    }
}

void animate_scenery(void) {
    // Called each tick from game loop
    for (i = 0; i < animated_scenery_object_count; ++i) {
        animate_object(animated_scenery_object_indexes[i]);
    }
}
```

**Animation Limit**: Only 20 scenery objects can animate simultaneously (performance constraint).

### 27.6 Destructible Scenery

```c
void damage_scenery(short object_index) {
    struct object_data *object = get_object_data(object_index);
    struct scenery_definition *definition = get_scenery_definition(object->permutation);

    if (definition->flags & _scenery_can_be_destroyed) {
        // Change to destroyed appearance
        object->shape = definition->destroyed_shape;

        // Spawn destruction effect (sparks, explosion)
        new_effect(&object->location, object->polygon,
            definition->destroyed_effect, object->facing);

        // No longer special (becomes normal debris)
        SET_OBJECT_OWNER(object, _object_is_normal);
    }
}
```

**Destruction Flow**:
```
┌──────────────────┐     damage_scenery()     ┌──────────────────┐
│ Intact Scenery   │ ───────────────────────► │ Destroyed State  │
│                  │                          │                  │
│ ┌──┐  Hanging    │                          │  ╲╱  Broken      │
│ │██│   Light     │                          │  ──   Light      │
│ └──┘             │                          │                  │
│                  │    + _effect_lamp_       │                  │
│ _scenery_can_be_ │      _breaking           │ _object_is_      │
│ _destroyed       │                          │ _normal          │
└──────────────────┘                          └──────────────────┘
```

### 27.7 Scenery Dimensions

```c
void get_scenery_dimensions(
    short scenery_type,
    world_distance *radius,
    world_distance *height)
{
    struct scenery_definition *definition = get_scenery_definition(scenery_type);

    *radius = definition->radius;
    *height = definition->height;
}
```

**Height Convention**:
- Positive height: Floor-standing object
- Negative height: Ceiling-hanging object (lights, decorations)

```
Ceiling ────────────────────────────────
              ┌─┐
              │█│  height = -WORLD_ONE/8
              └─┘  (hanging)

         ┌────────┐
         │████████│  height = WORLD_ONE_HALF
         │████████│  (standing)
Floor ───┴────────┴─────────────────────
```

---

## 28. Computer Terminal System

Marathon's terminals are interactive information displays that deliver story content, objectives, and allow teleportation between levels. The terminal system uses a custom scripting language preprocessed into efficient binary format.

### 28.1 Terminal Architecture

**Source Files**: `computer_interface.c`, `computer_interface.h`

```
Terminal Interaction Flow:
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Player activates    enter_computer_    Terminal state        │
│   control panel  ───► interface()    ───► machine starts        │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              TERMINAL STATE MACHINE                      │   │
│   │                                                          │   │
│   │   _reading_terminal ──► Groups processed sequentially    │   │
│   │         │                                                │   │
│   │         ▼                                                │   │
│   │   ┌──────────┐   ┌──────────┐   ┌──────────┐            │   │
│   │   │  LOGON   │──►│ CONTENT  │──►│  LOGOFF  │            │   │
│   │   │  Screen  │   │ (groups) │   │ /Teleport│            │   │
│   │   └──────────┘   └──────────┘   └──────────┘            │   │
│   │                                                          │   │
│   │   _no_terminal_state ──► Player exits terminal mode      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 28.2 Terminal Scripting Language

Terminals are authored using a simple markup language that gets preprocessed:

**Group Commands** (define terminal sections):
| Command | Purpose | Parameter |
|---------|---------|-----------|
| `#LOGON XXXX` | Login screen | Shape ID for login graphic |
| `#LOGOFF` | Logout screen | None |
| `#UNFINISHED` | Show if mission incomplete | None |
| `#SUCCESS` | Show if mission complete | None |
| `#FAILURE` | Show if mission failed | None |
| `#INFORMATION` | General text info | None |
| `#CHECKPOINT XX` | Show map checkpoint | Checkpoint/goal ID |
| `#PICT XXXX` | Display image | PICT resource ID |
| `#SOUND XXXX` | Play sound effect | Sound ID |
| `#MOVIE XXXX` | Play QuickTime movie | Movie ID |
| `#TRACK XXXX` | Play music track | Track ID |
| `#INTERLEVEL TELEPORT XXX` | Go to another level | Level number |
| `#INTRALEVEL TELEPORT XXX` | Teleport within level | Polygon index |
| `#STATIC XX` | Display static effect | Duration in ticks |
| `#CAMERA XX` | Show camera view | Object index |
| `#TAG XX` | Activate tagged objects | Tag number |
| `#END` | End current group | None |

**Text Formatting Codes** (embedded in text):
| Code | Effect |
|------|--------|
| `$B` | Bold ON |
| `$b` | Bold OFF |
| `$I` | Italic ON |
| `$i` | Italic OFF |
| `$U` | Underline ON |
| `$u` | Underline OFF |
| `$$` | Literal `$` character |

**Example Terminal Script**:
```
#LOGON 1234
Welcome to Terminal 47.
#END

#UNFINISHED
$BObjective:$b Find the primary reactor.

The pfhor have overrun this section. Proceed with
$Iextreme$i caution.
#CHECKPOINT 3
#END

#SUCCESS
$BWell done.$b The reactor is secured.

Proceed to the extraction point.
#INTERLEVEL TELEPORT 5
#END
```

### 28.3 Preprocessed Terminal Format

Terminal text is compiled into an efficient binary format stored in map WADs:

**Static Header** (`struct static_preprocessed_terminal_data`):
```c
struct static_preprocessed_terminal_data {
    short total_length;      // Total bytes of terminal data
    short flags;             // _text_is_encoded_flag (0x0001)
    short lines_per_page;    // Lines visible per screen
    short grouping_count;    // Number of terminal_groupings
    short font_changes_count; // Number of text_face_data entries
};
```

**Binary Layout**:
```
┌─────────────────────────────────────────────────────────────────┐
│  static_preprocessed_terminal_data (10 bytes)                   │
├─────────────────────────────────────────────────────────────────┤
│  terminal_groupings[grouping_count] (12 bytes each)             │
│    ├─ Group 0: type, permutation, start_index, length          │
│    ├─ Group 1: ...                                              │
│    └─ Group N: ...                                              │
├─────────────────────────────────────────────────────────────────┤
│  text_face_data[font_changes_count] (6 bytes each)              │
│    ├─ Face 0: index, face, color                                │
│    └─ Face N: ...                                               │
├─────────────────────────────────────────────────────────────────┤
│  char text[] (raw text with formatting codes stripped)          │
└─────────────────────────────────────────────────────────────────┘
```

**Terminal Grouping Structure**:
```c
struct terminal_groupings {
    short flags;              // _draw_object_on_right, _center_object
    short type;               // Group type enum (see below)
    short permutation;        // Type-specific parameter
    short start_index;        // Offset into text array
    short length;             // Text length for this group
    short maximum_line_count; // Lines in this group
};
```

**Group Types**:
```c
enum {
    _logon_group,              // 0 - Login screen
    _unfinished_group,         // 1 - Mission incomplete
    _success_group,            // 2 - Mission complete
    _failure_group,            // 3 - Mission failed
    _information_group,        // 4 - General info
    _end_group,                // 5 - End marker
    _interlevel_teleport_group,// 6 - Level transition
    _intralevel_teleport_group,// 7 - In-level teleport
    _checkpoint_group,         // 8 - Map checkpoint display
    _sound_group,              // 9 - Play sound
    _movie_group,              // 10 - Play movie
    _track_group,              // 11 - Play music
    _pict_group,               // 12 - Display image
    _logoff_group,             // 13 - Logout screen
    _camera_group,             // 14 - Camera view
    _static_group,             // 15 - TV static effect
    _tag_group                 // 16 - Activate tagged items
};
```

### 28.4 Text Face Data

Font style changes are stored separately for efficient rendering:

```c
struct text_face_data {
    short index;   // Character position where style changes
    short face;    // Style flags (see below)
    short color;   // Text color index
};
```

**Face Flags**:
```c
enum {
    _plain_text     = 0x00,
    _bold_text      = 0x01,
    _italic_text    = 0x02,
    _underline_text = 0x04
};
```

**Style Change Example**:
```
Text: "This is $Bbold$b text"

After preprocessing:
  text = "This is bold text"

  text_face_data[0] = { index: 0,  face: _plain_text,  color: 0 }
  text_face_data[1] = { index: 8,  face: _bold_text,   color: 0 }
  text_face_data[2] = { index: 12, face: _plain_text,  color: 0 }
```

### 28.5 Terminal Navigation

**Key Bindings** (action flags):
| Key | Action |
|-----|--------|
| Arrow Up / Page Up | `_terminal_page_up` - Scroll up |
| Arrow Down / Page Down | `_terminal_page_down` - Scroll down |
| Tab / Enter / Return / Space | `_terminal_next_state` - Next group |
| Escape | `_any_abort_key_mask` - Exit terminal |

**Player Terminal State**:
```c
struct player_terminal_data {
    short flags;                  // _terminal_is_dirty
    short phase;                  // Animation/timing phase
    short state;                  // _reading_terminal or _no_terminal_state
    short current_group;          // Which group being displayed
    short level_completion_state; // For choosing success/failure
    short current_line;           // Scroll position
    short maximum_line;           // Total lines in current group
    short terminal_id;            // Terminal being accessed
    long last_action_flag;        // For debouncing input
};
```

### 28.6 Terminal Rendering

**Display Layout**:
```
┌─────────────────────────────────────────────────────────────────┐
│                         BORDER (18px)                           │
├─────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │                                                             │ │
│ │   Marathon Terminal Network                                 │ │
│ │   ─────────────────────────                                │ │
│ │                                                             │ │
│ │   $BObjective:$b Secure the reactor                        │ │
│ │                                                             │ │
│ │   The pfhor have breached containment.                     │ │
│ │   Proceed to sublevel 3.                                   │ │
│ │                                           ┌───────────────┐ │ │
│ │                                           │  CHECKPOINT   │ │ │
│ │                                           │     MAP       │ │ │
│ │                                           │    (goal)     │ │ │
│ │                                           └───────────────┘ │ │
│ │                                                             │ │
│ └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│                    [Press SPACE to continue]                    │
└─────────────────────────────────────────────────────────────────┘
```

**Rendering Constants**:
```c
#define BORDER_HEIGHT 18
#define BORDER_INSET 9
#define LABEL_INSET 3
#define LOG_DURATION_BEFORE_TIMEOUT (2*TICKS_PER_SECOND)  // 60 ticks
#define MAXIMUM_FACE_CHANGES_PER_TEXT_GROUPING 128
```

### 28.7 Level Completion Detection

Terminals check mission state to show appropriate content:

```c
// In enter_computer_interface():
short completion_flag;  // Passed from control panel

// Determines which groups to show:
// completion_flag == 0: Show _unfinished_group
// completion_flag == 1: Show _success_group
// completion_flag == 2: Show _failure_group
```

**Completion Flow**:
```
Player activates terminal ───► Check mission goals
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              Goals incomplete    Goals complete    Goals failed
                    │                 │                 │
                    ▼                 ▼                 ▼
              #UNFINISHED         #SUCCESS          #FAILURE
              groups shown        groups shown      groups shown
```

---

## 29. Music/Soundtrack System

Marathon's music system provides background soundtrack with introduction/chorus/trailer structure, automatic looping, and fade transitions.

### 29.1 Music Architecture

**Source Files**: `music.c`, `music.h`, `song_definitions.h`

**State Machine**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    MUSIC STATE MACHINE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   _no_song_playing                                              │
│          │                                                      │
│          │ queue_song(index)                                    │
│          ▼                                                      │
│   _delaying_for_loop ──────────────────────┐                   │
│          │                                  │                   │
│          │ delay expires                    │                   │
│          ▼                                  │                   │
│   _playing_introduction                     │                   │
│          │                                  │                   │
│          │ intro complete                   │                   │
│          ▼                                  │                   │
│   _playing_chorus ◄────────────────────────┐│                   │
│          │                      │          ││                   │
│          │                      │ loop     ││                   │
│          │ chorus count done    └──────────┘│                   │
│          ▼                                  │                   │
│   _playing_trailer                          │                   │
│          │                                  │                   │
│          │ trailer complete                 │                   │
│          ▼                                  │                   │
│   _song_completed flag ─────────────────────┘                   │
│          │                     (if _song_automatically_loops)   │
│          │                                                      │
│          ▼ (if no loop)                                         │
│   _no_song_playing                                              │
│                                                                 │
│   ┌───────────────────────────────────────────────────────┐    │
│   │ fade_out_music() can transition to:                   │    │
│   │   _music_fading ───► gradual volume decrease         │    │
│   │                 ───► _no_song_playing when done      │    │
│   └───────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 29.2 Music Data Structure

```c
struct music_data {
    boolean initialized;          // Handler ready
    short flags;                  // _song_completed, _song_paused
    short state;                  // Current playback state
    short phase;                  // Timing counter
    short fade_duration;          // Total fade time
    short play_count;             // Chorus repetitions
    short song_index;             // Current song
    short next_song_index;        // Queued song (or NONE)
    short song_file_refnum;       // File handle to Music file
    short fade_interval_duration; // Ticks between volume steps
    short fade_interval_ticks;    // Counter for volume steps
    long ticks_at_last_update;    // For delta timing
    char *sound_buffer;           // Playback buffer
    long sound_buffer_size;       // Buffer size (500KB default)
    SndChannelPtr channel;        // Audio channel
    FilePlayCompletionUPP completion_proc;  // Callback
};
```

### 29.3 Song Definition

Each song has structured sections for varied playback:

```c
struct song_definition {
    short flags;                    // _song_automatically_loops
    long sound_start;               // File offset to song data
    struct sound_snippet introduction;  // Intro section
    struct sound_snippet chorus;    // Main loop section
    short chorus_count;             // Times to play chorus (negative = random)
    struct sound_snippet trailer;   // Outro section
    long restart_delay;             // Ticks before looping
};

struct sound_snippet {
    long start_offset;   // Byte offset in file
    long end_offset;     // End byte offset
};
```

**Song Structure Visualization**:
```
Song File Layout:
┌─────────────────────────────────────────────────────────────────┐
│ │◄── Introduction ──►│◄───── Chorus ─────►│◄── Trailer ──►│    │
│ ├────────────────────┼────────────────────┼────────────────┤    │
│ │    intro.start     │   chorus.start     │  trailer.start │    │
│ │         ↓          │        ↓           │       ↓        │    │
│ │    intro.end       │   chorus.end       │  trailer.end   │    │
└─────────────────────────────────────────────────────────────────┘

Playback Order:
  Introduction → Chorus (×N) → Trailer → [restart_delay] → Loop
                    ▲                                        │
                    └────────────────────────────────────────┘
                         (if _song_automatically_loops)
```

### 29.4 Music API

```c
// Initialize music system with song file
boolean initialize_music_handler(FileDesc *song_file);

// Queue a song to play (fades out current if playing)
void queue_song(short song_index);

// Fade out over duration (in ticks)
void fade_out_music(short duration);

// Stop immediately
void stop_music(void);

// Pause/resume
void pause_music(boolean pause);

// Check if music is playing
boolean music_playing(void);

// Called each tick to update state machine
void music_idle_proc(void);

// Release audio channel (for other sound needs)
void free_music_channel(void);
```

### 29.5 Fade System

**Fade Out Process**:
```c
#define BUILD_STEREO_VOLUME(l, r) ((((long)(r))<<16)|(l))

void fade_out_music(short duration) {
    music_state->fade_duration = duration;
    music_state->phase = duration;
    music_state->state = _music_fading;
    music_state->fade_interval_duration = 5;  // Volume steps every 5 ticks
    music_state->fade_interval_ticks = 5;
    music_state->song_index = NONE;  // Or next song to play
}

// In music_idle_proc(), _music_fading state:
// new_volume = (0x100 * phase) / fade_duration
// Ranges from 256 (full) to 0 (silent)
```

**Fade Visualization**:
```
Volume
  256 ┤▓▓▓▓▓▓▓▓▓▓▓
      │          ▓▓▓▓
      │              ▓▓▓▓
  128 ┤                  ▓▓▓▓
      │                      ▓▓▓▓
      │                          ▓▓▓▓
    0 ┼────────────────────────────────────────
      0    fade_duration/2     fade_duration
                 Time (ticks)
```

### 29.6 Music Constants

```c
#define kDefaultSoundBufferSize (500*KILO)  // 500KB buffer
#define NUMBER_OF_SONGS  // Defined by song_definitions.h

enum {  // Music states
    _no_song_playing,
    _playing_introduction,
    _playing_chorus,
    _playing_trailer,
    _delaying_for_loop,
    _music_fading
};

enum {  // Music flags
    _no_flags         = 0x0000,
    _song_completed   = 0x0001,
    _song_paused      = 0x0002
};

enum {  // Song flags
    _no_song_flags          = 0x0000,
    _song_automatically_loops = 0x0001
};
```

---

## 30. Error Handling & Progress Display

### 30.1 Game Error System

**Source Files**: `game_errors.c`, `game_errors.h`

Marathon uses a simple global error state for propagating errors:

```c
// Error types
enum {
    systemError,   // OS-level errors
    gameError,     // Game-specific errors
    NUMBER_OF_TYPES
};

// API
void set_game_error(short type, short error_code);
short get_game_error(short *type);
boolean error_pending(void);
void clear_game_error(void);
```

**Error Flow**:
```
Operation fails ───► set_game_error(type, code)
                              │
                              ▼
Caller checks ◄──── error_pending() returns TRUE
                              │
                              ▼
Handle error  ◄──── get_game_error(&type)
                              │
                              ▼
Continue      ◄──── clear_game_error()
```

### 30.2 Progress Display

**Source Files**: `progress.c`, `progress.h`

Progress dialogs show loading status during map/resource loading:

```c
struct progress_data {
    DialogPtr dialog;        // Mac dialog window
    GrafPtr old_port;        // Saved graphics port
    UserItemUPP progress_bar_upp;  // Custom draw procedure
};

// API
void open_progress_dialog(short message_id);
void set_progress_dialog_message(short message_id);
void draw_progress_bar(long sent, long total);
void close_progress_dialog(void);
```

**Progress Bar Visualization**:
```
┌─────────────────────────────────────────┐
│                                         │
│   Loading shapes...                     │
│                                         │
│   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░  │
│   ├──── sent ────┤                      │
│   ├─────────── total ─────────────────┤ │
│                                         │
└─────────────────────────────────────────┘

// Bar width calculation:
width = (sent * RECTANGLE_WIDTH(&bounds)) / total;
```

---

## 31. Resource Forks: Complete Guide

This section provides comprehensive documentation on Mac resource forks as they relate to Marathon's file formats.

### 31.1 What Are Resource Forks?

Classic Macintosh files have two parts:

```
Classic Mac File Structure:
┌─────────────────────────────────────────────────────────────────┐
│                        FILE                                     │
├─────────────────────────────┬───────────────────────────────────┤
│       DATA FORK             │         RESOURCE FORK             │
│                             │                                   │
│ • Sequential byte stream    │ • Structured database             │
│ • Like normal files on      │ • Contains typed resources        │
│   other platforms           │ • Each resource has:              │
│ • Standard fopen/fread      │   - 4-char type code              │
│                             │   - Numeric ID                    │
│                             │   - Optional name                 │
│                             │   - Data blob                     │
└─────────────────────────────┴───────────────────────────────────┘
```

### 31.2 Marathon's File Types Summary

**CRITICAL: Most Marathon files DON'T need resource fork access!**

| File | Format | Needs Resource Fork? | Notes |
|------|--------|---------------------|-------|
| **Shapes16** | Binary data fork | ✗ NO | Standard fopen/fread works |
| **Shapes8** | Binary data fork | ✗ NO | Standard fopen/fread works |
| **Sounds16** | Binary data fork | ✗ NO | Standard fopen/fread works |
| **Sounds8** | Binary data fork | ✗ NO | Standard fopen/fread works |
| **Map files** | Marathon WAD | ✗ NO | Standard fopen/fread works |
| **Saved games** | Marathon WAD | ✗ NO | Standard fopen/fread works |
| **Images** | Resource fork | ✓ YES | PICT resources (optional) |
| **Scenario files** | Resource fork | ✓ YES | Optional custom content |
| **Music** | Resource fork | ✓ YES | Optional, can skip |

### 31.3 The Extraction History

Understanding why there's confusion about Marathon's file formats:

```
1994: Marathon 1 Released
      └─► Shapes/Sounds stored in RESOURCE FORKS
          (Mac-only format)

1995: Marathon 2 Released
      └─► Bungie created extraction tools:
          ├─► shapeextract.c: Resource fork → Data fork binary
          └─► sndextract.c: Resource fork → Data fork binary

      └─► Retail Marathon 2 shipped with EXTRACTED files
          (Data fork binary format - cross-platform readable!)

2000/2011: Source Code Released
      └─► Includes both:
          ├─► Extraction tools (for historical reference)
          └─► Code to read extracted data fork files

          └─► Comment in shapes_macintosh.c:53 says
              "open the resource fork" but it's MISLEADING
              It actually opens the DATA FORK!
```

### 31.4 Why You Don't Need Resource Forks

For a basic Marathon port, you need:
- ✓ Shapes16 (textures/sprites) - **Data fork binary**
- ✓ Sounds16 (sound effects) - **Data fork binary**
- ✓ Map files (levels) - **Marathon WAD format**

All of these are readable with standard C file I/O:
```c
FILE* fp = fopen("Shapes16", "rb");
fread(buffer, size, 1, fp);
fclose(fp);
```

### 31.5 Resource Fork Structure (Reference)

If you do need to read resource forks (for Images file), here's the format:

**Resource Fork Layout**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    RESOURCE FORK STRUCTURE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Offset 0-255:     Reserved (256 bytes)                          │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Resource Data Section                                       │ │
│ │   ├─► Resource 1 data blob                                  │ │
│ │   ├─► Resource 2 data blob                                  │ │
│ │   └─► ...                                                   │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ Resource Map                                                │ │
│ │   ├─► Map Header (28 bytes)                                 │ │
│ │   │     └─► data_offset, map_offset, data_length, etc.     │ │
│ │   ├─► Type List                                             │ │
│ │   │     └─► Count of resource types                         │ │
│ │   │     └─► Type entries: 'PICT', 'snd ', 'clut', etc.     │ │
│ │   ├─► Reference List (per type)                             │ │
│ │   │     └─► Resource ID, name offset, data offset          │ │
│ │   └─► Name List                                             │ │
│ │         └─► Pascal strings for named resources              │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Resource Map Header**:
```c
struct resource_map_header {
    uint32_t data_offset;       // Offset to resource data
    uint32_t map_offset;        // Offset to this map
    uint32_t data_length;       // Length of data section
    uint32_t map_length;        // Length of map section
    // ... additional fields
};
```

**Type List Entry**:
```c
struct resource_type_entry {
    char type[4];              // e.g., 'PICT', 'snd '
    uint16_t count_minus_one;  // Number of resources - 1
    uint16_t reference_offset; // Offset to reference list
};
```

### 31.6 Accessing Resource Forks on Modern Systems

**On macOS** (resource forks still supported):
```bash
# Access via special path suffix
cp "Images/..namedfork/rsrc" images_rsrc.bin

# Or using xattr
xattr -p com.apple.ResourceFork Images > images_rsrc.bin
```

**Using Python** (cross-platform):
```python
# Install: pip install macresources
import macresources

with open("Images/..namedfork/rsrc", "rb") as f:
    resources = macresources.parse_file(f.read())

    # Extract all PICT resources
    for res in resources[b'PICT']:
        with open(f"pict_{res.id}.bin", "wb") as out:
            out.write(res.data)
```

**Using DeRez** (macOS developer tools):
```bash
# Decompile to text format
DeRez -only PICT "Images" > images_pict.r
```

### 31.7 Practical Strategy for Porting

**Recommended approach**:

```
Option 1: Skip Images File (Fastest)
├─► Stub interface graphics with colored rectangles
├─► Focus on core gameplay
└─► Add graphics later if needed

Option 2: One-Time Extraction (Recommended)
├─► On macOS, extract PICT resources once
├─► Convert to PNG/BMP using ImageMagick
├─► Bundle converted images with your port
└─► Load with stb_image or similar

Option 3: Use Aleph One Assets
├─► Aleph One has already converted everything
├─► Download their data files
├─► Reference their conversion scripts
└─► Most complete solution
```

**Extraction Script Example**:
```python
#!/usr/bin/env python3
"""Extract Marathon Images file PICTs to PNG."""

import os
import subprocess
from pathlib import Path

def extract_images(images_path, output_dir):
    # Read resource fork
    rsrc_path = f"{images_path}/..namedfork/rsrc"

    # Use macresources library
    import macresources
    with open(rsrc_path, "rb") as f:
        resources = macresources.parse_file(f.read())

    # Extract each PICT
    os.makedirs(output_dir, exist_ok=True)
    for res in resources.get(b'PICT', []):
        pict_path = f"{output_dir}/pict_{res.id}.pict"
        png_path = f"{output_dir}/pict_{res.id}.png"

        # Write raw PICT
        with open(pict_path, "wb") as f:
            f.write(res.data)

        # Convert to PNG using ImageMagick
        subprocess.run(["convert", pict_path, png_path])
        os.remove(pict_path)

        print(f"Extracted PICT {res.id}")

if __name__ == "__main__":
    extract_images("Images", "extracted_images/")
```

### 31.8 Resource Types in Marathon's Images File

| Type Code | Description | Usage |
|-----------|-------------|-------|
| `PICT` | QuickDraw picture | Interface graphics, logos |
| `clut` | Color lookup table | Palette data |
| `cicn` | Color icon | UI icons |
| `CURS` | Cursor | Mouse cursors |

### 31.9 Summary Decision Tree

```
Do I need to parse resource forks?

START
  │
  ▼
Are you porting just core gameplay?
  │
  ├─► YES: You DON'T need resource forks!
  │        Shapes, Sounds, Maps are all data fork files.
  │
  └─► NO: Do you need interface graphics?
          │
          ├─► NO: You don't need resource forks.
          │
          └─► YES: Choose extraction method:
                   │
                   ├─► Use Aleph One's pre-converted assets
                   ├─► Extract once on macOS, convert to PNG
                   └─► Write resource fork parser (complex)
```

---

## 32. Life of a Frame: Complete Pipeline

This section provides a comprehensive walkthrough of what happens during a single frame of Marathon gameplay—from reading player input to pixels appearing on screen. Understanding this pipeline is essential for porting, debugging, and optimizing the engine.

### Overview: The Three Phases

Every frame follows this fundamental sequence:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        LIFE OF A FRAME                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌────────────┐      ┌────────────┐      ┌────────────┐                    │
│   │   INPUT    │ ──►  │   UPDATE   │ ──►  │   RENDER   │                    │
│   │  (60 Hz)   │      │  (30 Hz)   │      │ (Variable) │                    │
│   └────────────┘      └────────────┘      └────────────┘                    │
│        │                    │                    │                           │
│        │                    │                    │                           │
│   parse_keymap()      update_world()       render_screen()                  │
│   action_flags        tick_count++         render_view()                    │
│   queue flags         physics, AI          texture_map()                    │
│                                                                              │
│   Time budget:        Time budget:         Time budget:                     │
│   ~1ms                ~5-15ms              ~15-25ms                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Insight**: Input is captured at 60 Hz (VBL rate), but game logic runs at a fixed 30 Hz. Rendering runs as fast as possible, potentially faster than 30 FPS on capable hardware.

---

### Phase 1: Input Collection

Input is collected via the vertical blank interrupt (VBL) at 60 Hz—twice per game tick.

#### Input Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INPUT COLLECTION                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Hardware              parse_keymap()             Action Queue              │
│   ─────────             ──────────────             ────────────              │
│                                                                              │
│   ┌─────────┐          ┌──────────────────┐       ┌──────────────┐          │
│   │Keyboard │──────►   │ Read GetKeys()   │       │ Player 0     │          │
│   └─────────┘          │ Check modifiers  │       │ action_flags │          │
│                        │ Build flags      │       │ [queue]      │          │
│   ┌─────────┐          └────────┬─────────┘       └──────────────┘          │
│   │ Mouse   │──────►            │                                           │
│   │ Δx, Δy  │                   │                 ┌──────────────┐          │
│   │ buttons │                   ▼                 │ Player 1     │          │
│   └─────────┘          ┌──────────────────┐       │ action_flags │          │
│                        │process_action_   │──────►│ [queue]      │          │
│   ┌─────────┐          │flags(player_idx, │       └──────────────┘          │
│   │ Keypad  │──────►   │  &flags, count)  │                                 │
│   │(weapon) │          └──────────────────┘       ┌──────────────┐          │
│   └─────────┘                   │                 │ Player N     │          │
│                                 │                 │ action_flags │          │
│                                 ▼                 │ [queue]      │          │
│                        heartbeat_count++          └──────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Action Flags

All player input is encoded into a 32-bit `action_flags` bitmask:

| Bit(s) | Flag Name | Meaning |
|--------|-----------|---------|
| 0 | `_moving_forward` | W or Up arrow |
| 1 | `_moving_backward` | S or Down arrow |
| 2 | `_turning_left` | A or Left arrow |
| 3 | `_turning_right` | D or Right arrow |
| 4 | `_sidestepping_left` | Alt+Left or Q |
| 5 | `_sidestepping_right` | Alt+Right or E |
| 6 | `_looking_up` | Page Up |
| 7 | `_looking_down` | Page Down |
| 8 | `_action_trigger` | Tab (use switches/terminals) |
| 9 | `_left_trigger` | Primary fire (mouse button) |
| 10 | `_right_trigger` | Secondary fire (option+click) |
| 11 | `_sidestep_dont_turn` | Alt modifier active |
| 12 | `_look_dont_turn` | Look modifier active |
| 13 | `_toggle_map` | M key |
| 14 | `_microphone_button` | Network talk |
| 15 | `_swim` | Swim up when in liquid |
| 16-31 | Various | Weapon switch, absolute yaw/pitch/position |

#### Key Source Locations

```c
// vbl.c - parse_keymap() builds action_flags from raw input
long action_flags= parse_keymap();  // Read keyboard + mouse
process_action_flags(local_player_index, &action_flags, 1);  // Queue for player
heartbeat_count++;  // VBL counter increments

// Action queue is per-player circular buffer
// Allows decoupling input rate (60Hz) from game tick rate (30Hz)
// Critical for network synchronization
```

---

### Phase 2: World Update

The world updates at a fixed 30 ticks per second, processing all game systems in a specific order.

#### Update Sequence Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        update_world() SEQUENCE                               │
│                        [marathon2.c:73-149]                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   for (i = 0; i < time_elapsed; ++i)  // May process multiple ticks         │
│   {                                                                          │
│       ┌─────────────────────────────────────────────────────────────┐       │
│       │                ENVIRONMENT UPDATES                           │       │
│       ├─────────────────────────────────────────────────────────────┤       │
│       │  1. update_lights()        - Animate light sources          │       │
│       │  2. update_medias()        - Update liquid heights/flow     │       │
│       │  3. update_platforms()     - Move elevators/doors           │       │
│       └─────────────────────────────────────────────────────────────┘       │
│                              │                                               │
│                              ▼                                               │
│       ┌─────────────────────────────────────────────────────────────┐       │
│       │                INTERACTIVE UPDATES                           │       │
│       ├─────────────────────────────────────────────────────────────┤       │
│       │  4. update_control_panels() - Check switch timeouts         │       │
│       │  5. update_players()        - Apply action_flags, physics   │       │
│       └─────────────────────────────────────────────────────────────┘       │
│                              │                                               │
│                              ▼                                               │
│       ┌─────────────────────────────────────────────────────────────┐       │
│       │                COMBAT & PHYSICS                              │       │
│       ├─────────────────────────────────────────────────────────────┤       │
│       │  6. move_projectiles()     - Advance bullets/grenades       │       │
│       │  7. move_monsters()        - AI decisions, pathfinding      │       │
│       │  8. update_effects()       - Explosions, debris, decals     │       │
│       └─────────────────────────────────────────────────────────────┘       │
│                              │                                               │
│                              ▼                                               │
│       ┌─────────────────────────────────────────────────────────────┐       │
│       │                WORLD MAINTENANCE                             │       │
│       ├─────────────────────────────────────────────────────────────┤       │
│       │  9. recreate_objects()      - Respawn items in netgames     │       │
│       │ 10. handle_random_sound_image() - Ambient sounds            │       │
│       │ 11. animate_scenery()       - Cycle scenery frames          │       │
│       │ 12. update_net_game()       - Network game rules            │       │
│       └─────────────────────────────────────────────────────────────┘       │
│                              │                                               │
│                              ▼                                               │
│       ┌─────────────────────────────────────────────────────────────┐       │
│       │                GAME STATE                                    │       │
│       ├─────────────────────────────────────────────────────────────┤       │
│       │ 13. check_level_change()   - Teleport to next level?        │       │
│       │ 14. game_is_over()         - Check victory/death            │       │
│       │ 15. tick_count++           - Advance world clock            │       │
│       │ 16. game_time_remaining--  - Countdown timer                │       │
│       └─────────────────────────────────────────────────────────────┘       │
│   }                                                                          │
│                                                                              │
│   if (time_elapsed) {                                                        │
│       update_interface()   - Update HUD state                               │
│       update_fades()       - Screen transition effects                      │
│   }                                                                          │
│   check_recording_replaying()  - Demo playback                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Update Order Rationale

The order is carefully designed for correct simulation:

| Order | System | Why This Order? |
|-------|--------|-----------------|
| 1-3 | Environment | Must update before entities that depend on environment state |
| 4 | Control Panels | Must update before players process actions |
| 5 | Players | Player physics uses current platform heights |
| 6 | Projectiles | Move before monsters so hit detection is current |
| 7 | Monsters | AI reacts to current projectile/player positions |
| 8 | Effects | Visual effects follow physics resolution |
| 9-12 | Maintenance | Order doesn't matter (independent) |
| 13-14 | State | Must be last to catch level changes from any system |

#### Player Update Detail

```c
// player.c - update_players() processes each player:
for (player_index = 0; player_index < dynamic_world->player_count; player_index++)
{
    // 1. Get action flags from queue
    action_flags = get_action_queue_entry(player_index);

    // 2. Apply physics
    update_player_physics_variables(player_index, action_flags);

    // 3. Update weapons
    update_player_weapons(player_index, action_flags);

    // 4. Handle action trigger (switches, terminals)
    if (action_flags & _action_trigger)
        player_touched_control_panel(player_index);

    // 5. Handle damage, oxygen, powerups
    update_player_state(player_index);
}
```

---

### Phase 3: Rendering

After the world updates, the frame is rendered. This is where data becomes pixels.

#### Rendering Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        RENDERING PIPELINE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   render_screen()                                                            │
│   [screen.c:668]                                                            │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  1. SETUP VIEW                                                  │        │
│   │     - Copy player position/facing to world_view                 │        │
│   │     - Set tick_count, shading_mode (normal/infravision)         │        │
│   │     - Handle extravision (fisheye) field-of-view changes        │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  2. CHECK SPECIAL MODES                                         │        │
│   │     - Overhead map active? → render_overhead_map()              │        │
│   │     - Terminal active? → render_computer_interface()            │        │
│   │     - Otherwise → render_view() (3D world)                      │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   render_view()                                                              │
│   [render.c:497]                                                            │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  3. BUILD RENDER TREE (Visibility)                              │        │
│   │     build_render_tree(view)                                     │        │
│   │     - Initialize root with player's polygon                     │        │
│   │     - Cast rays at view frustum edges (left_edge, right_edge)   │        │
│   │     - Flood-fill through portals via cast_render_ray()          │        │
│   │     - Build tree of node_data with portal clipping info         │        │
│   │     Output: Tree of ~50-100 visible polygons from ~500-1000     │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  4. SORT RENDER TREE (Depth Ordering)                           │        │
│   │     sort_render_tree(view)                                      │        │
│   │     - Convert node_data tree to sorted_node_data list           │        │
│   │     - Sort back-to-front by distance (painter's algorithm)      │        │
│   │     - Build clipping_window_data for each sorted node           │        │
│   │     Output: Sorted list with clipping windows attached          │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  5. BUILD OBJECT LIST (Sprites)                                 │        │
│   │     build_render_object_list(view)                              │        │
│   │     - Walk sorted nodes, collect objects from each polygon      │        │
│   │     - Classify as interior (one polygon) or exterior (spans)    │        │
│   │     - Build aggregate clipping windows for exterior objects     │        │
│   │     Output: render_object_data list sorted into node tree       │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  6. RENDER TREE (Draw Geometry)                                 │        │
│   │     render_tree(view, destination)                              │        │
│   │     For each sorted node (back to front):                       │        │
│   │       - render_node_floor_or_ceiling() if above/below viewer    │        │
│   │       - render_node_side() for each visible wall                │        │
│   │       - render_node_object() for sprites in this polygon        │        │
│   │     Output: Pixels to framebuffer                               │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  7. RENDER WEAPONS (Viewer Layer)                               │        │
│   │     render_viewer_sprite_layer(view, destination)               │        │
│   │     - Draw held weapon sprites over 3D view                     │        │
│   │     - Handle weapon animation frames                            │        │
│   │     Output: Final 3D scene complete                             │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   Back to render_screen():                                                  │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │  8. DRAW HUD                                                    │        │
│   │     draw_panels() - Health, oxygen, ammo, motion sensor         │        │
│   │     Output: Complete frame ready for display                    │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   DISPLAY (blit to screen)                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### render_tree() Detail: Surface Rendering Order

Within each polygon, surfaces are rendered in this order:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              PER-POLYGON RENDERING ORDER                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   For each visible polygon (back to front):                                 │
│                                                                              │
│   ┌───────────────────────────────────────────────┐                         │
│   │ 1. CEILING (if visible - player looking up)   │                         │
│   │    render_node_floor_or_ceiling()             │                         │
│   │    Uses AFFINE texture mapping (horizontal)   │                         │
│   └───────────────────────────────────────────────┘                         │
│                          │                                                   │
│                          ▼                                                   │
│   ┌───────────────────────────────────────────────┐                         │
│   │ 2. WALLS (all visible sides)                  │                         │
│   │    render_node_side()                         │                         │
│   │    For each side:                             │                         │
│   │      - _full_side: single texture             │                         │
│   │      - _high_side: upper portion only         │                         │
│   │      - _low_side: lower portion only          │                         │
│   │      - _split_side: both upper and lower      │                         │
│   │      - transparent texture (if present)       │                         │
│   │    Uses PERSPECTIVE texture mapping (vertical)│                         │
│   └───────────────────────────────────────────────┘                         │
│                          │                                                   │
│                          ▼                                                   │
│   ┌───────────────────────────────────────────────┐                         │
│   │ 3. FLOOR (if visible - player looking down)   │                         │
│   │    render_node_floor_or_ceiling()             │                         │
│   │    Uses AFFINE texture mapping (horizontal)   │                         │
│   │    May be replaced by media (water/lava)      │                         │
│   └───────────────────────────────────────────────┘                         │
│                          │                                                   │
│                          ▼                                                   │
│   ┌───────────────────────────────────────────────┐                         │
│   │ 4. EXTERIOR OBJECTS (sprites in this polygon) │                         │
│   │    render_node_object()                       │                         │
│   │    Monsters, items, projectiles, effects      │                         │
│   └───────────────────────────────────────────────┘                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Rendering Deep Dive: Visibility and Clipping

The rendering pipeline uses several key data structures that work together:

**Key Data Structures:**

| Structure | Purpose | Key Fields |
|-----------|---------|------------|
| `node_data` | Raw visibility tree node | `polygon_index`, `children`, `clipping_endpoints[]`, `clipping_lines[]` |
| `sorted_node_data` | Depth-sorted node ready for rendering | `polygon_index`, `clipping_windows`, `interior_objects`, `exterior_objects` |
| `clipping_window_data` | Screen region visible through portal chain | `x0, x1, y0, y1` (screen bounds), `left, right, top, bottom` (clip vectors), `next_window` |
| `render_object_data` | Sprite ready for rendering | `node`, `clipping_windows`, `rectangle`, `ymedia` |

**build_render_tree() - Portal Flood Fill:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VISIBILITY TREE CONSTRUCTION                              │
│                    build_render_tree() [render.c:702]                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. Initialize with player's polygon as root node                          │
│                                                                              │
│   2. Cast initial rays at view frustum edges:                               │
│      cast_render_ray(view, &left_edge, NONE, root, _counterclockwise_bias)  │
│      cast_render_ray(view, &right_edge, NONE, root, _clockwise_bias)        │
│                                                                              │
│   3. Process polygon queue (flood-fill through portals):                    │
│      ┌─────────────────────────────────────────────────────────────┐        │
│      │  while (polygon_queue not empty):                           │        │
│      │    polygon = dequeue()                                      │        │
│      │    for each vertex in polygon:                              │        │
│      │      if vertex not yet visited:                             │        │
│      │        calculate_endpoint_clipping_information()            │        │
│      │        cast_render_ray() toward vertex                      │        │
│      │          → may discover new polygons through portals        │        │
│      │          → add children to current node                     │        │
│      │          → queue newly visible polygons                     │        │
│      └─────────────────────────────────────────────────────────────┘        │
│                                                                              │
│   Output: Tree of node_data with parent/child relationships                 │
│           representing portal visibility                                    │
│                                                                              │
│   Example tree:                                                             │
│                                                                              │
│         [Polygon 42]  ← Player's polygon (root)                             │
│              │                                                              │
│        ┌─────┴─────┐                                                        │
│        ▼           ▼                                                        │
│   [Polygon 17] [Polygon 23]  ← Visible through portals                      │
│        │                                                                    │
│        ▼                                                                    │
│   [Polygon 8]  ← Visible through Polygon 17's portal                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Clipping Windows - How Portal Occlusion Works:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLIPPING WINDOW ACCUMULATION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   As visibility traverses through portals, each portal RESTRICTS            │
│   what can be seen. Clipping windows track these restrictions.              │
│                                                                              │
│   Screen View:                                                              │
│   ┌─────────────────────────────────────────────────────┐                   │
│   │                                                     │                   │
│   │    ┌──────────────────────────────┐                │                   │
│   │    │   Portal A (first portal)    │                │                   │
│   │    │    ┌────────────────┐        │                │                   │
│   │    │    │ Portal B       │        │                │                   │
│   │    │    │ (seen through A)        │                │                   │
│   │    │    │   ┌────────┐   │        │                │                   │
│   │    │    │   │Polygon │   │        │                │                   │
│   │    │    │   │ seen   │   │        │                │                   │
│   │    │    │   │through │   │        │                │                   │
│   │    │    │   │A and B │   │        │                │                   │
│   │    │    │   └────────┘   │        │                │                   │
│   │    │    └────────────────┘        │                │                   │
│   │    └──────────────────────────────┘                │                   │
│   │                                                     │                   │
│   └─────────────────────────────────────────────────────┘                   │
│                                                                              │
│   Each polygon may have MULTIPLE clipping windows (linked list)             │
│   if it's visible through multiple portal chains.                           │
│                                                                              │
│   struct clipping_window_data {                                             │
│       world_vector2d left, right;   // Horizontal clip vectors              │
│       world_vector2d top, bottom;   // Vertical clip vectors (j=k)          │
│       short x0, x1, y0, y1;         // Screen pixel bounds                  │
│       struct clipping_window_data *next_window;  // Linked list             │
│   };                                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Interior vs Exterior Objects:**

| Type | Definition | Rendering |
|------|------------|-----------|
| **Interior** | Sprite fits entirely within one polygon | Rendered with that polygon's clipping window |
| **Exterior** | Sprite spans multiple polygons | Gets aggregate clipping window from all base nodes |

Objects are sorted into the render tree based on which polygon(s) contain them. Exterior objects (large sprites spanning portals) require special handling to build a combined clipping window from all polygons they intersect.

**Actual Texture Mapping Functions:**

| Function | Used For | Technique |
|----------|----------|-----------|
| `texture_horizontal_polygon()` | Floors, ceilings | Affine mapping (assumes horizontal surface) |
| `texture_vertical_polygon()` | Walls, sides | Perspective-correct mapping |
| `texture_rectangle()` | Sprites, objects, weapons | Scaled rectangle with clipping |

**Transfer Modes (Special Effects):**

| Mode | Value | Effect |
|------|-------|--------|
| `_xfer_normal` | 0 | Standard texture mapping |
| `_xfer_fade_out_to_black` | 1 | Darkens toward black (distance fade) |
| `_xfer_invisibility` | 2 | Partial transparency |
| `_xfer_subtle_invisibility` | 3 | Nearly invisible shimmer |
| `_xfer_pulsate` | 4 | Brightness oscillates |
| `_xfer_wobble` | 5 | Texture distortion |
| `_xfer_fast_wobble` | 6 | Rapid distortion |
| `_xfer_static` | 7 | Random noise (like TV static) |
| `_xfer_landscape` | 9 | Horizon mapping for skies |
| `_xfer_smear` | 10 | Horizontal smear effect |
| `_xfer_fade_out_static` | 11 | Static with fade |
| `_xfer_tinted` | 15+ | Color tinting |

#### Texture Mapping: From Surface to Pixels

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              TEXTURE MAPPING PIPELINE                                        │
│              [scottish_textures.c]                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Surface Definition                    Texture Lookup                       │
│   ──────────────────                    ──────────────                       │
│                                                                              │
│   ┌─────────────────┐                  ┌─────────────────┐                  │
│   │ screen x, y     │                  │ texture u, v    │                  │
│   │ depth (z)       │ ───────────────► │ (from surface   │                  │
│   │ texture coords  │   calculate      │  mapping)       │                  │
│   │ light level     │   u,v at pixel   │                 │                  │
│   └─────────────────┘                  └────────┬────────┘                  │
│                                                 │                            │
│                                                 ▼                            │
│                                        ┌─────────────────┐                  │
│                                        │ Fetch texel     │                  │
│                                        │ (8-bit index)   │                  │
│                                        │ from bitmap     │                  │
│                                        └────────┬────────┘                  │
│                                                 │                            │
│                                                 ▼                            │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                    SHADING TABLE LOOKUP                          │       │
│   │                                                                  │       │
│   │   shade_index = depth_to_shade[distance >> SHADE_SHIFT]          │       │
│   │   final_color = shading_table[shade_index][texel]                │       │
│   │                                                                  │       │
│   │   ┌─────────────────────────────────────────────────────┐       │       │
│   │   │ Shading Table (one per light level)                  │       │       │
│   │   │                                                      │       │       │
│   │   │   shade 0 (close/bright):  [256 color lookups]      │       │       │
│   │   │   shade 1:                 [256 color lookups]      │       │       │
│   │   │   shade 2:                 [256 color lookups]      │       │       │
│   │   │   ...                                               │       │       │
│   │   │   shade 31 (far/dark):     [256 color lookups]      │       │       │
│   │   │                                                      │       │       │
│   │   │   texel 0x42 at shade 5 → color 0x3B                │       │       │
│   │   └─────────────────────────────────────────────────────┘       │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                                 │                            │
│                                                 ▼                            │
│                                        ┌─────────────────┐                  │
│                                        │ Write pixel to  │                  │
│                                        │ framebuffer     │                  │
│                                        │ (8/16/32 bit)   │                  │
│                                        └─────────────────┘                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Complete Frame Timeline

Here's the complete timeline showing all three phases:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     COMPLETE FRAME TIMELINE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Time (ms)  0      8      16      24      33      41      50      58       │
│              │      │       │       │       │       │       │       │       │
│              │      │       │       │       │       │       │       │       │
│   VBL (60Hz) ▼      ▼       ▼       ▼       ▼       ▼       ▼       ▼       │
│              ●      ●       ●       ●       ●       ●       ●       ●       │
│              │      │       │       │       │       │       │       │       │
│   INPUT:     ├──────┤       │       ├───────┤       │       ├───────┤       │
│   collect    │parse │       │       │ parse │       │       │ parse │       │
│   action     │keymap│       │       │ keymap│       │       │ keymap│       │
│   flags      │queue │       │       │ queue │       │       │ queue │       │
│              │      │       │       │       │       │       │       │       │
│   TICK       ├──────────────┤       ├───────────────┤       ├───────────────│
│   (30Hz)     │  TICK N      │       │  TICK N+1     │       │  TICK N+2     │
│              │              │       │               │       │               │
│   UPDATE:    │ ┌──────────┐ │       │ ┌───────────┐ │       │ ┌───────────┐ │
│              │ │update    │ │       │ │ update    │ │       │ │ update    │ │
│              │ │_world()  │ │       │ │ _world()  │ │       │ │ _world()  │ │
│              │ │ lights   │ │       │ │ lights    │ │       │ │ lights    │ │
│              │ │ platforms│ │       │ │ platforms │ │       │ │ platforms │ │
│              │ │ players  │ │       │ │ players   │ │       │ │ players   │ │
│              │ │ monsters │ │       │ │ monsters  │ │       │ │ monsters  │ │
│              │ │ etc...   │ │       │ │ etc...    │ │       │ │ etc...    │ │
│              │ └──────────┘ │       │ └───────────┘ │       │ └───────────┘ │
│              │              │       │               │       │               │
│   RENDER:    │  ┌──────────────┐    │  ┌───────────────┐    │  ┌───────────│
│              │  │render_screen │    │  │render_screen  │    │  │render     │
│              │  │ build tree   │    │  │ build tree    │    │  │ ...       │
│              │  │ sort nodes   │    │  │ sort nodes    │    │  │           │
│              │  │ draw surfaces│    │  │ draw surfaces │    │  │           │
│              │  │ draw sprites │    │  │ draw sprites  │    │  │           │
│              │  │ draw HUD     │    │  │ draw HUD      │    │  │           │
│              │  └──────────────┘    │  └───────────────┘    │  │           │
│              │              │       │               │       │               │
│   DISPLAY:   │              ●       │               ●       │               │
│              │         (blit)       │          (blit)       │               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Data Flow: From Keypress to Pixel

A complete trace of a single action (firing a weapon):

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           DATA FLOW: PLAYER FIRES WEAPON                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   USER ACTION: Player clicks mouse button                                    │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │ INPUT PHASE                                                     │        │
│   │   GetMouse() → mouse button down                               │        │
│   │   parse_keymap() → sets _left_trigger bit in action_flags      │        │
│   │   process_action_flags() → queues for local player             │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │ UPDATE PHASE                                                    │        │
│   │   update_players():                                            │        │
│   │     action_flags = dequeue player flags                        │        │
│   │     if (action_flags & _left_trigger):                         │        │
│   │       update_player_weapons() →                                │        │
│   │         fire_weapon() →                                        │        │
│   │           new_projectile() →                                   │        │
│   │             Creates projectile object in world                 │        │
│   │             Sets position, velocity, damage                    │        │
│   │           play_weapon_sound()                                  │        │
│   │           set_weapon_animation(_firing)                        │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │ RENDER PHASE                                                    │        │
│   │   render_screen():                                             │        │
│   │     world_view->tick_count updated                             │        │
│   │                                                                 │        │
│   │   render_view():                                               │        │
│   │     build_render_tree() →                                      │        │
│   │       Projectile's polygon added to visibility                 │        │
│   │     build_render_object_list() →                               │        │
│   │       Projectile sprite added to render list                   │        │
│   │     render_tree() →                                            │        │
│   │       render_node_object() draws projectile sprite             │        │
│   │     render_viewer_sprite_layer() →                             │        │
│   │       Draws weapon in firing animation frame                   │        │
│   │       get_weapon_display_information() returns muzzle flash    │        │
│   └────────────────────────────────────────────────────────────────┘        │
│        │                                                                     │
│        ▼                                                                     │
│   ┌────────────────────────────────────────────────────────────────┐        │
│   │ PIXELS ON SCREEN                                                │        │
│   │   - Weapon sprite shows muzzle flash                           │        │
│   │   - Projectile sprite visible (if player can see it)           │        │
│   │   - Sound plays through audio system                           │        │
│   └────────────────────────────────────────────────────────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Timing Budget

Typical frame time breakdown for Marathon on target hardware (1995 Macintosh):

| Phase | Typical Time | Percentage | Notes |
|-------|--------------|------------|-------|
| Input | 0.5-1 ms | ~3% | VBL-driven, very fast |
| Update (per tick) | 5-15 ms | ~30% | Monster AI is expensive |
| Render | 15-25 ms | ~60% | Texture mapping dominates |
| Display/Blit | 1-3 ms | ~7% | Memory copy to screen |
| **Total** | **22-44 ms** | **100%** | **23-45 FPS** |

On modern hardware (2020+):

| Phase | Typical Time | Notes |
|-------|--------------|-------|
| Input | < 0.1 ms | Trivial |
| Update | 0.5-2 ms | Even complex AI is fast |
| Render | 1-5 ms | Software rendering still works |
| **Total** | **2-8 ms** | **120-500 FPS** (capped to 30 ticks) |

---

### Main Loop Code Reference

The main game loop in `interface.c`:

```c
// interface.c - idle_game_state() lines 536-545
if (game_state.state == _game_in_progress)
{
    // Only process if keyboard controller active (not paused)
    if (get_keyboard_controller_status() && (ticks_elapsed = update_world()))
    {
        // World updated successfully, render the results
        render_screen(ticks_elapsed);
    }
    else if (no_frame_rate_limit)
    {
        // Render even without update (for high refresh displays)
        render_screen(0);
    }
}
```

Key observations:
1. `get_keyboard_controller_status()` returns FALSE during pause/menu
2. `update_world()` returns number of ticks processed (0 if queue empty)
3. `render_screen()` called only after successful update (or with frame skip)
4. `ticks_elapsed` can be > 1 if frame rate drops below 30 FPS

---

### Summary Tables

#### Phase Comparison

| Aspect | Input | Update | Render |
|--------|-------|--------|--------|
| **Rate** | 60 Hz (VBL) | 30 Hz (fixed) | Variable |
| **Trigger** | Interrupt | Main loop | After update |
| **Duration** | ~1 ms | 5-15 ms | 15-25 ms |
| **State** | Reads hardware | Modifies world | Reads world |
| **Deterministic** | Yes | Yes | N/A |
| **Network sync** | Queued | Critical | Not synced |

#### Key Functions by Phase

| Phase | Primary Function | Location | Purpose |
|-------|-----------------|----------|---------|
| Input | `parse_keymap()` | vbl_macintosh.c:200 | Read keyboard/mouse |
| Input | `process_action_flags()` | vbl.c:376 | Queue for player |
| Update | `update_world()` | marathon2.c:73 | Advance simulation |
| Update | `update_players()` | player.c:384 | Apply player input |
| Update | `move_monsters()` | monsters.c:280 | AI and pathfinding |
| Render | `render_screen()` | screen.c:668 | Setup and dispatch |
| Render | `render_view()` | render.c:497 | 3D world rendering |
| Render | `build_render_tree()` | render.c:702 | Portal visibility flood-fill |
| Render | `sort_render_tree()` | render.c:1116 | Depth sort + clipping windows |
| Render | `render_tree()` | render.c:2182 | Draw sorted polygons |
| Render | `texture_horizontal_polygon()` | scottish_textures.c:277 | Floor/ceiling mapping |
| Render | `texture_vertical_polygon()` | scottish_textures.c:476 | Wall mapping |
| Render | `texture_rectangle()` | scottish_textures.c:665 | Sprite/object mapping |

---

## Appendix A: Glossary of Terms

### Units and Measurements

| Term | Definition |
|------|------------|
| **World Unit** | Base measurement unit. `WORLD_ONE = 1024`. Approximately 2 meters in real scale. |
| **Fixed-Point** | 16.16 format integer math. `FIXED_ONE = 65536`. Used for precise calculations without floating-point. |
| **Tick** | One game logic update. 30 ticks = 1 second. Physics, AI, and game state advance per tick. |
| **Angle** | Direction in 512ths of a circle. `FULL_CIRCLE = 512`. 90° = 128 angle units. |

### Geometry

| Term | Definition |
|------|------------|
| **Polygon** | Convex floor/ceiling region. Building block of Marathon maps. Contains 3-8 vertices. |
| **Line** | Edge connecting two endpoints. May be solid (wall) or portal (passable). |
| **Side** | Wall surface on one side of a line. Contains texture/lighting data. |
| **Endpoint** | Vertex position in 2D map space. Shared by multiple lines. |
| **Portal** | Transparent line connecting two polygons. Enables room-over-room geometry. |

### Rendering

| Term | Definition |
|------|------------|
| **Clipping Window** | Screen region visible through a portal. Restricts what's drawn behind. |
| **Render Node** | Entry in visibility tree. One per visible polygon. |
| **Shading Table** | 256-entry lookup for distance-based darkening. Pre-computed per lighting level. |
| **Transfer Mode** | Special rendering effect: normal, tinted, static, landscape, etc. |
| **Shape Descriptor** | 16-bit ID combining collection (5 bits) + CLUT (3 bits) + shape index (8 bits). |

### Collections and Shapes

| Term | Definition |
|------|------------|
| **Collection** | Set of related graphics (one monster type, one environment's walls, etc.). |
| **CLUT** | Color Look-Up Table. Palette variant within a collection (lighting levels). |
| **High-Level Shape** | Animation sequence (e.g., "walking"). Contains views × frames. |
| **Low-Level Shape** | Single frame with positioning data. Points to bitmap. |
| **Bitmap** | Actual pixel data. May be row-order or column-order, raw or RLE-compressed. |

### Entities

| Term | Definition |
|------|------------|
| **Monster** | AI-controlled entity. Uses physics model for movement. |
| **Projectile** | Fired object (bullet, rocket, energy bolt). Moves each tick until impact. |
| **Effect** | Temporary visual (explosion, splash, spark). Auto-removes when animation ends. |
| **Scenery** | Static decoration. May be solid, animated, or destructible. |
| **Item** | Pickup object (weapon, ammo, health). Collected on player contact. |
| **Platform** | Moving floor/ceiling. Used for doors, elevators, crushers. |

### Physics

| Term | Definition |
|------|------------|
| **Action Flags** | Bitmask of player inputs for one tick. Deterministic for networking. |
| **External Velocity** | Movement from outside forces (explosions, currents). Decays over time. |
| **Angular Velocity** | Turning speed. Applied to facing angle each tick. |
| **Elevation** | Vertical look angle. Pitch up/down from horizontal. |
| **Step Delta** | Vertical camera offset from walking. Creates view bobbing. |

### Media (Liquids)

| Term | Definition |
|------|------------|
| **Media** | Liquid surface in a polygon (water, lava, etc.). Has height, flow, damage. |
| **Submerged Fade** | Screen color tint when player's head is below media surface. |
| **Media Detonation** | Splash effect when projectile hits liquid surface. |

### Files and Data

| Term | Definition |
|------|------------|
| **WAD** | Marathon's tagged archive format. NOT related to Doom WADs. |
| **Tag** | 4-character identifier for data type in WAD file (e.g., 'POLY', 'LITE'). |
| **Data Fork** | Standard file content (readable on all platforms). Shapes/Sounds use this. |
| **Resource Fork** | Mac-specific structured data. Only used for Images file. |

### Networking

| Term | Definition |
|------|------------|
| **Ring Protocol** | Token-passing network topology. Each player sends to next in ring. |
| **Action Queue** | Buffer of future action flags. Allows latency compensation. |
| **Deterministic** | Same inputs = same results. Required for lock-step multiplayer. |
| **Sync** | Periodic state verification between players. Detects desynchronization. |

### Sound

| Term | Definition |
|------|------------|
| **Ambient Sound** | Looping environmental audio attached to polygon. |
| **Random Sound** | Intermittent environmental sound. Probability-based playback. |
| **Obstruction** | Sound volume reduction from walls between source and listener. |
| **Permutation** | Sound variation. Random selection from multiple recordings. |

---

## Appendix B: Quick Reference Card

### Fixed-Point Mathematics

| Constant | Value | Description |
|----------|-------|-------------|
| `FIXED_ONE` | 65536 (1<<16) | 1.0 in 16.16 fixed-point |
| `FIXED_FRACTIONAL_BITS` | 16 | Bits after decimal point |
| `WORLD_ONE` | 1024 (1<<10) | 1.0 in world units |
| `WORLD_FRACTIONAL_BITS` | 10 | Bits after decimal point |
| `WORLD_ONE_HALF` | 512 | 0.5 world units |
| `WORLD_ONE_FOURTH` | 256 | 0.25 world units |

**Conversions**:
```c
INTEGER_TO_FIXED(x)  = (x) << 16
FIXED_TO_INTEGER(x)  = (x) >> 16
WORLD_TO_FIXED(x)    = (x) << 6
FIXED_TO_WORLD(x)    = (x) >> 6
```

### Angles

| Constant | Value | Degrees |
|----------|-------|---------|
| `NUMBER_OF_ANGLES` | 512 | 360° |
| `FULL_CIRCLE` | 512 | 360° |
| `HALF_CIRCLE` | 256 | 180° |
| `QUARTER_CIRCLE` | 128 | 90° |
| `EIGHTH_CIRCLE` | 64 | 45° |
| `SIXTEENTH_CIRCLE` | 32 | 22.5° |

**Conversion**: `degrees = (angle × 360) / 512`

### Trigonometry

| Constant | Value | Description |
|----------|-------|-------------|
| `TRIG_SHIFT` | 10 | Shift for trig results |
| `TRIG_MAGNITUDE` | 1024 | Trig table scale factor |
| `ANGULAR_BITS` | 9 | Bits used for angles |

**Usage**: `result = (distance × cosine_table[angle]) >> TRIG_SHIFT`

### Timing

| Constant | Value | Description |
|----------|-------|-------------|
| `TICKS_PER_SECOND` | 30 | Game logic rate |
| `TICKS_PER_MINUTE` | 1800 | 30 × 60 |
| `MACHINE_TICKS_PER_SECOND` | 60 | Mac VBL rate |

### Rendering Limits

| Constant | Value | Description |
|----------|-------|-------------|
| `MAXIMUM_NODES` | 512 | Portal tree nodes |
| `MAXIMUM_SORTED_NODES` | 128 | Rendered polygons |
| `MAXIMUM_RENDER_OBJECTS` | 72 | Visible sprites |
| `MAXIMUM_CLIPPING_WINDOWS` | 256 | Clipping regions |
| `NORMAL_FIELD_OF_VIEW` | 80 | Degrees |
| `EXTRAVISION_FIELD_OF_VIEW` | 130 | With powerup |

### World Limits

| Constant | Value | Description |
|----------|-------|-------------|
| `MAXIMUM_POLYGONS_PER_MAP` | 1024 | Max polygons |
| `MAXIMUM_SIDES_PER_MAP` | 4096 | Max wall sides |
| `MAXIMUM_LINES_PER_MAP` | 4096 | Max line segments |
| `MAXIMUM_ENDPOINTS_PER_MAP` | 8192 | Max vertices |
| `MAXIMUM_LIGHTS` | 64 | Max light sources |
| `MAXIMUM_PLATFORMS` | 64 | Max elevators/doors |
| `MAXIMUM_SAVED_OBJECTS` | 384 | Map object placements |

### Entity Limits

| Constant | Value | Description |
|----------|-------|-------------|
| `MAXIMUM_MONSTERS_PER_MAP` | 220 | Active monsters |
| `MAXIMUM_PROJECTILES_PER_MAP` | 32 | Active projectiles |
| `MAXIMUM_OBJECTS_PER_MAP` | 384 | Total objects |
| `MAXIMUM_EFFECTS` | 64 | Visual effects |
| `MAXIMUM_PLAYERS` | 8 | Network players |

### Physics Constants

| Constant | Value | Real Value | Description |
|----------|-------|------------|-------------|
| `GRAVITATIONAL_ACCELERATION` | FIXED_ONE/400 | 0.0025/tick | Gravity |
| `TERMINAL_VELOCITY` | FIXED_ONE/7 | ~0.143/tick | Max fall speed |
| `COEFFICIENT_OF_ABSORBTION` | 2 | 0.25× | Bounce damping |

**Walking Model**:
| Constant | Value | Description |
|----------|-------|-------------|
| Max Forward Velocity | FIXED_ONE/14 | ~0.071/tick |
| Max Backward Velocity | FIXED_ONE/17 | ~0.059/tick |
| Max Strafe Velocity | FIXED_ONE/20 | 0.050/tick |
| Acceleration | FIXED_ONE/200 | 0.005/tick² |
| Deceleration | FIXED_ONE/100 | 0.010/tick² |

### Collision Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MINIMUM_SEPARATION_FROM_WALL` | WORLD_ONE/4 | 256 units |
| `MAXIMUM_STEP_HEIGHT` | WORLD_ONE/3 | ~341 units |
| `MAXIMUM_ARM_REACH` | 3×WORLD_ONE/4 | ~768 units |

### Sound Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MAXIMUM_SOUND_VOLUME` | 256 | Full volume |
| `MAXIMUM_SOUND_CHANNELS` | 4 | Normal channels |
| `MAXIMUM_AMBIENT_SOUND_CHANNELS` | 2 | Background channels |

### Automap Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `OVERHEAD_MAP_MINIMUM_SCALE` | 1 | Zoomed out |
| `OVERHEAD_MAP_MAXIMUM_SCALE` | 4 | Zoomed in |
| `DEFAULT_OVERHEAD_MAP_SCALE` | 3 | Default zoom |

### Motion Sensor Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MAXIMUM_MOTION_SENSOR_ENTITIES` | 12 | Tracked blips |
| `MOTION_SENSOR_RANGE` | 8×WORLD_ONE | Detection radius |
| `NUMBER_OF_PREVIOUS_LOCATIONS` | 6 | Trail length |

### Common Type Sizes

| Type | Size | Description |
|------|------|-------------|
| `world_point2d` | 4 bytes | 2D position |
| `world_point3d` | 6 bytes | 3D position |
| `fixed_point3d` | 12 bytes | High-precision 3D |
| `endpoint_data` | 16 bytes | Vertex with metadata |
| `line_data` | 32 bytes | Line segment |
| `side_data` | 64 bytes | Wall surface |
| `polygon_data` | 128 bytes | Room/area |
| `monster_definition` | 128 bytes | Monster template |
| `weapon_definition` | 196 bytes | Weapon template |

---

## Appendix C: Source File Index

### Core Engine

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Main Loop** | `shell.c`, `shell.h` | Game shell and main loop |
| **Game State** | `interface.c`, `interface.h` | State machine (menu, game, etc.) |
| **World Update** | `world.c`, `world.h` | Per-tick world updates |

### World Representation

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Map Structures** | `map.h` | Core map data structures |
| **Map Operations** | `map.c` | Map queries and manipulation |
| **Platforms** | `platforms.c`, `platforms.h` | Elevators, doors |
| **Media (Liquids)** | `media.c`, `media.h` | Water, lava, goo |
| **Lights** | `lightsource.c`, `lightsource.h` | Dynamic lighting |
| **Flood Fill/Zones** | `flood_map.c`, `flood_map.h` | Zone calculations |

### Rendering

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Render Pipeline** | `render.c`, `render.h` | Main 3D rendering |
| **Texture Mapping** | `scottish_textures.c`, `scottish_textures.h` | Software texture mapper |
| **Low-Level Textures** | `low_level_textures.c` | Texture utilities |
| **Screen Management** | `screen.c`, `screen.h` | Display output |
| **Fades/Effects** | `fades.c`, `fades.h` | Screen color effects |
| **Overhead Map** | `overhead_map.c`, `overhead_map.h` | Automap rendering |
| **Shapes/Sprites** | `shapes.c`, `shapes.h` | Shape collection management |
| **Textures** | `textures.c`, `textures.h` | Texture management |

### Physics & Collision

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Physics Core** | `physics.c`, `physics.h` | Movement and physics |
| **Physics Models** | `physics_models.h` | Physics constants |
| **Collision** | `map.c` (collision functions) | Wall/object collision |

### Entities

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Monsters** | `monsters.c`, `monsters.h` | Monster AI and behavior |
| **Monster Definitions** | `monster_definitions.h` | 47 monster types |
| **Weapons** | `weapons.c`, `weapons.h` | Weapon system |
| **Weapon Definitions** | `weapon_definitions.h` | Weapon stats |
| **Projectiles** | `projectiles.c`, `projectiles.h` | Bullet/missile physics |
| **Projectile Definitions** | `projectile_definitions.h` | Projectile types |
| **Effects** | `effects.c`, `effects.h` | Visual effects |
| **Effect Definitions** | `effect_definitions.h` | Effect types |
| **Items** | `items.c`, `items.h` | Pickup items |
| **Item Definitions** | `item_definitions.h` | Item types |
| **Scenery** | `scenery.c`, `scenery.h` | Static objects |
| **Scenery Definitions** | `scenery_definitions.h` | Scenery types |
| **Pathfinding** | `pathfinding.c` | Monster navigation |

### Player

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Player State** | `player.c`, `player.h` | Player data and updates |
| **Motion Sensor** | `motion_sensor.c`, `motion_sensor.h` | Radar system |
| **Control Panels** | `player.c` (panel functions) | Switch/terminal interaction |

### Audio

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Sound Logic** | `game_sound.c`, `game_sound.h` | Sound playback logic |
| **Sound Platform** | `sound_macintosh.c` | Mac Sound Manager interface |
| **Sound Definitions** | `sound_definitions.h` | Sound effect IDs |
| **Ambient Sounds** | `ambient_sound.h` | Environmental audio |

### Networking

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Network Core** | `network.c`, `network.h` | Core networking |
| **Network Games** | `network_games.c`, `network_games.h` | Game type logic |
| **DDP Protocol** | `network_ddp.c` | Low-level networking |
| **Network Dialogs** | `network_dialogs.c` | Multiplayer UI |

### File I/O

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **WAD Format** | `wad.c`, `wad.h` | Marathon WAD file format |
| **Game WAD** | `game_wad.c`, `game_wad.h` | Level loading/saving |
| **Tags** | `tags.h` | WAD tag definitions |
| **Files Platform** | `files_macintosh.c` | Mac file I/O |
| **Shapes Platform** | `shapes_macintosh.c` | Shape file loading |

### Input

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **VBL/Input** | `vbl.c`, `vbl.h` | Input processing, timing |
| **VBL Platform** | `vbl_macintosh.c` | Mac input implementation |
| **Mouse** | `mouse.c`, `mouse.h` | Mouse handling |
| **Action Flags** | `vbl.h` | Input encoding |

### Interface

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Computer Terminal** | `computer_interface.c`, `computer_interface.h` | Terminal system |
| **Game Window** | `game_window.c`, `game_window.h` | HUD rendering |
| **Game Window Platform** | `game_window_macintosh.c` | Mac HUD implementation |
| **Screen Drawing** | `screen_drawing.c`, `screen_drawing.h` | 2D drawing utilities |

### Replay/Recording

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Replay System** | `vbl.c` (recording functions) | Action recording |
| **Replay Header** | `vbl.h` | Replay data structures |

### Utilities (cseries.lib)

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Core Types** | `cseries.h` | Basic types and macros |
| **Mac Extensions** | `macintosh_cseries.h` | Mac-specific types |
| **Byte Swapping** | `byte_swapping.c`, `byte_swapping.h` | Endianness utilities |
| **Checksums** | `checksum.c`, `checksum.h` | CRC calculations |
| **RLE Compression** | `rle.c`, `rle.h` | Run-length encoding |

### Assembly Optimizations

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Textures 68K** | `scottish_textures.a` | 68K texture inner loops |
| **Textures PPC** | `scottish_textures.s` | PowerPC texture inner loops |
| **Textures 16-bit** | `scottish_textures16.a` | 16-bit mode 68K |
| **Screen 68K** | `screen.a` | 68K screen blitting |
| **Math PPC** | `quadruple.s` | 64-bit math PPC |
| **Network 68K** | `network_listener.a` | Network interrupt handler |

### Definition Headers (Data Tables)

| File | Contents |
|------|----------|
| `monster_definitions.h` | 47 monster type definitions |
| `weapon_definitions.h` | Weapon stats and behavior |
| `projectile_definitions.h` | Projectile types and physics |
| `effect_definitions.h` | Visual effect definitions |
| `item_definitions.h` | Pickup item definitions |
| `platform_definitions.h` | Platform/door type definitions |
| `sound_definitions.h` | Sound effect ID mappings |
| `scenery_definitions.h` | Scenery object types |

### Extraction Tools

| Topic | Primary Files | Description |
|-------|---------------|-------------|
| **Shape Extract** | `extract/shapeextract.c` | Extract shapes from resources |
| **Sound Extract** | `extract/sndextract.c` | Extract sounds from resources |

---

## Appendix D: Fixed-Point to Floating-Point Conversion (Optional Modernization)

This appendix discusses the **optional** conversion of Marathon's fixed-point math to floating-point. This is NOT recommended for initial porting but may be considered for future modernization.

### Why Marathon Uses Fixed-Point

**Historical Reasons (1994-1995)**:
- Many CPUs lacked FPUs (floating-point units)
- Integer math was 10-100× faster than software FP emulation
- Determinism for networked multiplayer was critical

**Technical Reasons**:
- Bit-identical results across platforms
- Predictable performance
- No NaN/Infinity edge cases

### Fixed-Point vs Floating-Point Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FIXED-POINT vs FLOATING-POINT                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  FIXED-POINT (16.16)               FLOATING-POINT (32-bit float)            │
│  ──────────────────                ─────────────────────────────             │
│                                                                              │
│  Range: -32768.0 to +32767.99998   Range: ±3.4×10³⁸ (huge!)                 │
│  Precision: 1/65536 ≈ 0.000015     Precision: ~7 significant digits         │
│  Operations: Integer ALU           Operations: FPU required                  │
│                                                                              │
│  Addition:    a + b (trivial)      a + b (trivial)                          │
│  Multiply:    (a * b) >> 16        a * b (trivial)                          │
│  Division:    (a << 16) / b        a / b (trivial, but slow)                │
│                                                                              │
│  Determinism: ✓ Perfect            ✗ May vary by CPU/compiler               │
│  Modern CPU:  ~1 cycle             ~1-5 cycles (with FPU)                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Conversion Complexity Analysis

| System | Complexity | Risk | Benefit |
|--------|------------|------|---------|
| **Rendering** | Medium | Low | Simplified texture math |
| **Physics** | High | Medium | Smoother movement |
| **Collision** | High | High | Potential precision edge cases |
| **Trigonometry** | Low | Low | Use standard sin/cos |
| **Networking** | **Critical** | **Very High** | Breaks determinism! |
| **Saved Games** | Medium | Medium | Format incompatibility |

### What Conversion Would Involve

**Step 1: Type Changes**
```c
// Before (fixed-point)
typedef long fixed;
typedef short world_distance;

// After (floating-point)
typedef float fixed;         // Or double for precision
typedef float world_distance;
```

**Step 2: Remove Shift Operations**
```c
// Before: Fixed-point multiply
result = (a * b) >> FIXED_FRACTIONAL_BITS;

// After: Float multiply
result = a * b;
```

**Step 3: Update Trigonometry**
```c
// Before: Table lookup
short angle = ...; // 0-511
fixed cos_val = cosine_table[angle];

// After: Standard library
float angle_rad = angle * (M_PI / 256.0f);
float cos_val = cosf(angle_rad);
```

**Step 4: Change Comparisons**
```c
// Before: Integer comparison
if (distance > WORLD_ONE)

// After: Float comparison with epsilon
#define EPSILON 0.0001f
if (distance > 1.0f - EPSILON)
```

### Files Requiring Changes

| File | Lines to Change | Difficulty |
|------|-----------------|------------|
| `cseries.h` | ~50 | Easy |
| `world.h` | ~100 | Medium |
| `physics.c` | ~800 | Hard |
| `render.c` | ~500 | Medium |
| `scottish_textures.c` | ~1200 | Hard |
| `monsters.c` | ~400 | Medium |
| `projectiles.c` | ~200 | Medium |
| `collision (in map.c)` | ~300 | Hard |
| **Total** | ~3500+ | **Significant** |

### Critical Warning: Networking Breaks!

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ⚠️ DETERMINISM WARNING ⚠️                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Marathon's multiplayer REQUIRES bit-identical simulation:                   │
│                                                                              │
│  Player 1 (Intel i7):     Player 2 (AMD Ryzen):                             │
│    sin(0.123) = 0.1226901...    sin(0.123) = 0.1226901...                   │
│                                                                              │
│  With different compilers, FPU modes, or optimizations:                      │
│    Player 1: 0.12269011139                                                   │
│    Player 2: 0.12269011140  ← DIFFERENT!                                    │
│                                                                              │
│  After 1000 ticks: positions diverge → desync → game breaks                 │
│                                                                              │
│  Solutions:                                                                  │
│  1. Use IEEE 754 strict mode (-ffp-contract=off, /fp:strict)                │
│  2. Force same rounding mode on all platforms                                │
│  3. Use software float library (slow)                                        │
│  4. Keep fixed-point for networked games (recommended)                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Recommendation

**For initial port**: Keep fixed-point math. It works, it's tested, and changes nothing.

**For single-player modernization**: Consider float conversion if:
- You want smoother physics
- You're not supporting multiplayer
- You're willing to break saved game compatibility

**For multiplayer**: Do NOT convert—or implement strict IEEE 754 compliance across all platforms.

### Other Future Modernization Options

| Modernization | Effort | Benefit | Trade-off |
|--------------|--------|---------|-----------|
| **GPU rendering** | Very High | Modern visuals, speed | Complete rewrite of renderer |
| **Higher resolution** | Low | Sharper display | Just change framebuffer size |
| **Widescreen** | Medium | Better FOV | Adjust view frustum calculations |
| **Higher tick rate** | High | Smoother gameplay | Physics tuning, network changes |
| **Modern audio** | Medium | Better sound | Already addressed in porting |
| **Mouse look** | Low | Modern controls | Already supported in source |
| **Texture filtering** | Medium | Smoother textures | Modify texture mapper |
| **Dynamic lighting** | High | Modern atmosphere | Add shadow mapping |
| **Particle systems** | Medium | Visual effects | Extend effects.c |

### Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FIXED-POINT CONVERSION DECISION TREE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Is this your first port?                                                    │
│      │                                                                       │
│      ├── YES ──► Keep fixed-point. Focus on getting it working first.       │
│      │                                                                       │
│      └── NO ───► Do you need multiplayer?                                   │
│                      │                                                       │
│                      ├── YES ──► Keep fixed-point for determinism.          │
│                      │                                                       │
│                      └── NO ───► Conversion is possible but significant.    │
│                                  Budget 40-80 hours of work.                 │
│                                  Test extensively for edge cases.            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Conclusion

The Marathon 2 & Infinity engine represents sophisticated 1990s software engineering, combining technical excellence with forward-thinking design principles that remain relevant today.

**Key Innovations**:
1. **Portal-based visibility** - More flexible than BSP for open spaces
2. **Fixed-point determinism** - Critical for networked multiplayer
3. **30 Hz fixed timestep** - Consistent physics across all systems
4. **Polygon-based world** - Explicit connectivity, O(1) traversal
5. **Deterministic networking** - Same inputs → same outputs
6. **Data-driven physics** - Multiple physics models without engine changes (Infinity)

**Technical Excellence**:
- Software rendering optimized for CPUs without GPUs
- Assembly-optimized critical paths
- Unified rendering pipeline (8/16/32-bit)
- Clean separation of concerns
- Platform-independent core (mostly)
- Modular design enabling community content

**Legacy**:
Marathon's techniques influenced later games and development practices:
- Halo (same studio, spiritual successor)
- Deterministic networking model widely adopted
- Fixed-timestep physics simulation (now industry standard)
- User-generated content ecosystem (pioneered by Infinity)
- Open-source game engines (Aleph One continues the tradition)

The dual source code releases—Marathon 2 in 2000 (GPL v2) and Marathon Infinity in 2011 (GPL v3)—were extraordinary gifts to the gaming community. The Infinity release, being complete and unredacted, has enabled projects like **Aleph One** (the modern open-source Marathon engine) and serves as an invaluable educational resource for game developers studying classic 3D engine architecture.

---

## References

- Marathon 2 & Infinity source code: `./marathon2/` and `./cseries.lib/` (this repository)
- Official source archive: https://infinitysource.bungie.org/
- Aleph One project: https://alephone.lhowon.org/
- Bungie: https://www.bungie.net/
- Licenses: Marathon 2 (GPL v2), Marathon Infinity (GPL v3)

---

**Acknowledgments**:
- Bungie Software - For creating Marathon and releasing both source code versions
- Aleph One Team - For maintaining and evolving the open-source engine
- Fabien Sanglard - For inspiring technical deep-dives with his Game Engine Black Books
- Marathon community - For decades of preservation, modding, and documentation

---

*This document represents comprehensive technical analysis of the Marathon engine based on the Marathon Infinity source code release (2011). All code examples are from the Marathon Infinity codebase (GPL v3), which represents the most complete and refined version of the Marathon 2 engine.*
