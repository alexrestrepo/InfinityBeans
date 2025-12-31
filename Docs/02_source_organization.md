# Chapter 2: Source Code Organization

## File Structure and Module Dependencies

---

## 2.1 Directory Structure

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
└── README.md               # Original source release notes
```

---

## 2.2 File Categories

### Core Game Logic (Platform-Independent)

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

### Mac-Specific Files (Must Replace)

These 11 files contain Macintosh-specific code and need platform replacements:

| File | Lines | Mac APIs Used | Replacement |
|------|-------|---------------|-------------|
| `files_macintosh.c` | ~430 | FSSpec, FSRead/Write | stdio |
| `shapes_macintosh.c` | ~660 | FSSpec, Handles | stdio + malloc |
| `sound_macintosh.c` | ~1,000 | Sound Manager | miniaudio |
| `vbl_macintosh.c` | ~400 | VBL, GetKeys | Fenster input |
| `interface_macintosh.c` | ~1,700 | Dialogs, Menus | Custom or stub |
| `game_window_macintosh.c` | ~220 | GrafPorts | Framebuffer |
| `overhead_map_macintosh.c` | ~200 | QuickDraw | Line drawing |
| `wad_macintosh.c` | ~150 | FSSpec paths | POSIX paths |
| `wad_prefs_macintosh.c` | ~200 | Preferences | Config file |
| `preprocess_map_mac.c` | ~150 | Resource fork | Not needed |
| `mouse.c` | ~200 | GetMouse | Fenster mouse |

### Assembly Files (Have C Fallbacks)

| File | Architecture | Purpose |
|------|--------------|---------|
| `scottish_textures.a` | 68K | Optimized texture loops |
| `scottish_textures.s` | PowerPC | Optimized texture loops |
| `screen.a` | 68K | Screen blitting |
| `quadruple.s` | PowerPC | 64-bit math |

**Note**: All assembly functions have C equivalents controlled by `#ifdef` blocks.

### Definition Headers (Data Tables)

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

---

## 2.3 Platform Abstraction Pattern

Marathon uses a consistent pattern for platform abstraction:

```
┌─────────────────────────────────────────────────────────────┐
│                    Game Logic Layer                          │
│  render.c, physics.c, monsters.c, weapons.c, etc.           │
│  (Platform-independent, uses abstract interfaces)            │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Abstract Interface                         │
│  shapes.c, game_sound.c, vbl.c, screen.c                    │
│  (Defines interface, may have some portable code)            │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Platform Implementation                      │
│  shapes_macintosh.c, sound_macintosh.c, vbl_macintosh.c     │
│  (Mac-specific code, replace for porting)                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2.4 The cseries.lib Foundation

The `cseries.lib` directory provides the base abstraction layer.

### Platform-Independent Definitions (cseries.h)

```c
typedef long fixed;           // 16.16 fixed-point
typedef unsigned short word;
typedef unsigned char byte;
typedef byte boolean;

#define FIXED_ONE (1<<16)     // 65536
#define TRUE 1
#define FALSE 0
#define NONE -1

// Memory management (redirects to platform layer)
void *new_pointer(long size);
void dispose_pointer(void *pointer);

// Debug support
void assert(expr);
void halt();
```

### Mac-Specific Extensions (macintosh_cseries.h)

```c
#include <Memory.h>
#include <QuickDraw.h>
#include <Events.h>

// Mac handle wrappers
Handle NewHandle(Size size);
void DisposeHandle(Handle h);
void HLock(Handle h);
void HUnlock(Handle h);
```

---

## 2.5 Key Abstractions to Replace

### Memory Management

```c
// Original (Mac handles)
Handle h = NewHandle(size);
HLock(h);
void* ptr = *h;
HUnlock(h);
DisposeHandle(h);

// Replacement (modern)
void* ptr = malloc(size);
free(ptr);
```

### File I/O

```c
// Original (Mac File Manager)
FSSpec spec;
FSMakeFSSpec(vRefNum, dirID, name, &spec);
FSpOpenDF(&spec, fsRdPerm, &refNum);
FSRead(refNum, &count, buffer);
FSClose(refNum);

// Replacement (stdio)
FILE* fp = fopen(path, "rb");
fread(buffer, 1, count, fp);
fclose(fp);
```

### Timing

```c
// Original (Mac Tick Count - 60 Hz)
unsigned long ticks = TickCount();

// Replacement
uint32_t ms = platform_get_ticks();  // milliseconds
```

---

## 2.6 Summary

Marathon's codebase has clear separation between portable and platform-specific code:

**Portable (~90%):**
- Core game logic (render, physics, AI)
- Data structures and definitions
- Game mechanics

**Platform-Specific (~10%):**
- 11 Mac-specific files to replace
- Assembly with C fallbacks
- Window/input/sound management

### Porting Strategy

1. Create platform abstraction layer
2. Replace 11 Mac-specific files
3. Use C fallbacks for assembly
4. Most code compiles unchanged

---

*Next: [Chapter 3: Engine Overview](03_engine_overview.md) - High-level architecture and subsystem interactions*
